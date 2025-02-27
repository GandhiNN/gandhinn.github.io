---
title:  "Migrating from Poetry to UV: An Experience"
seo_title: "migrating from poetry to uv: an experience"
seo_description: "migrating from poetry to uv: an experience"
date:   2025-02-26 00:00:00 +0700
categories:
  - Programming
tags:
  - Python
excerpt: "This post describes my experience migrating from Poetry to UV (+Ruff) as my package build framework."
---
### Background
TBC

### The Solution

#### The `Makefile`
{% highlight makefile %}
NAME := kumeza
# POETRY := $(shell command -v poetry 2> /dev/null)
UV := $(shell command -v uv 2> /dev/null)
RUFF := $(shell command -v ruff 2> /dev/null)

.DEFAULT_GOAL := help

# .PHONY: help
# help:
# 		@echo "Please use 'make <target> where <target> is one of:"
# 		@echo ""
# 		@echo " init				install prerequisites packages"
# 		@echo "	install				install packages and prepare environment"
# 		@echo "	install-poetry		install poetry package manager"
# 		@echo "	clean				remove all temporary files"
# 		@echo "	lint				run the code linters"
# 		@echo "	format				reformat code"
# 		@echo "	test 				run all the tests"
# 		@echo ""
# 		@echo "Check the Makefile to know exactly what each target is doing."

.PHONY: help
help:
		@echo "Please use 'make <target> where <target> is one of:"
		@echo ""
		@echo " init				install prerequisites packages"
		@echo "	install				install packages and prepare environment"
		@echo "	install-uv			install uv package manager"
		@echo "	clean				remove all temporary files"
		@echo "	lint				run the code linters"
		@echo "	format				reformat code"
		@echo "	test 				run all the tests"
		@echo ""
		@echo "Check the Makefile to know exactly what each target is doing."

.PHONY: init
init:
		python -m pip install --upgrade pip
		python -m pip install awscli
		python -m pip install git+https://github.com/awslabs/aws-glue-libs.git
		
# .PHONY: install-poetry
# install-poetry:
# 		@echo "Installing poetry..."
# 		pip install poetry
# 		@poetry --version

.PHONY: install
install:
		@echo "Installing uv..."
		pip install uv
		@echo "Installing ruff..."
		pip install ruff
		@ruff --version

# .PHONY: install
# install:
# 		@if [ -z $(POETRY) ]; then echo "Poetry could not be found. See https://python-poetry.org/docs/"; exit 2; fi
# 		$(POETRY) install

.PHONY: sync
sync:
		@if [ -z $(UV) ]; then echo "UV could not be found. See https://docs.astral.sh/uv/getting-started/installation/"; exit 2; fi
		$(UV) sync

# .PHONY: clean
# clean:
# 		find . -type d -name "__pycache__" | xargs rm -rf {};
# 		rm -rf .coverage .mypy_cache .pytest_cache ./dist ./htmlcov

.PHONY: clean
clean:
		find . -type d -name "__pycache__" | xargs rm -rf {};
		rm -rf .coverage .mypy_cache .pytest_cache ./dist ./htmlcov ./package

# .PHONY: format
# format: 
# 		$(POETRY) run isort --profile=black --lines-after-imports=2 ./tests/ $(NAME)
# 		$(POETRY) run black ./tests/ $(NAME)

.PHONY: format
format: 
		$(RUFF) format --verbose

# .PHONY: lint
# lint: 
# 		$(POETRY) run isort --profile=black --lines-after-imports=2 --check-only ./tests/ $(NAME)
# 		$(POETRY) run black --check ./tests/ $(NAME) --diff
# 		$(POETRY) run flake8 --ignore=W503,E501 ./tests/ $(NAME)
# 		$(POETRY) run mypy ./tests/ $(NAME) --ignore-missing-imports
# 		$(POETRY) run bandit -r $(NAME) -s B608

.PHONY: lint
lint: 
		$(RUFF) check

# .PHONY: test
# test: 
# 		$(POETRY) run pytest ./tests/ --cov-report term-missing --cov-fail-under 100 --cov $(NAME) --cov-report html

.PHONY: test
test: 
		$(UV) run pytest --cov $(NAME) --cov-report=term-missing --cov-report html --cov-fail-under 100 


# build:
# 		$(POETRY) build

build:
		$(UV) build

# lambda-layer:
# 		$(POETRY) install --only main --sync
# 		$(POETRY) build
# 		$(POETRY) run pip install --upgrade -t python dist/*.whl
# 		mkdir -p out; zip -r -q out/kumeza.zip python/* -i 'python/kumeza*' -x 'python/*.pyc'
# 		zip -r -q out/pyarrow.zip python/* -i 'python/*arrow*' -x 'python/*.pyc'

lambda-layer:
		$(UV) sync
		$(UV) build
		$(UV) run pip install --upgrade -t python dist/*.whl
		mkdir -p out; zip -r -q out/kumeza.zip python/* -i 'python/kumeza*' -x 'python/*.pyc'
		zip -r -q out/pyarrow.zip python/* -i 'python/*arrow*' -x 'python/*.pyc'
{% endhighlight %}

#### The `Jenkinsfile`
{% highlight jenkinsfile %}
// -----------------------------------------------------------------------------
// Variable Settings
// -----------------------------------------------------------------------------
ZIP_NAME = 'kumeza'
AWS_REGION = 'eu-west-1'
BB_PROJECT_NAME = 'daa1'
STREAM = 'daas'
DEEP_PRODUCT_NAME = 'icloud'

// Conditional switch for environment
if (env.BRANCH_NAME == "master") {
    DEEP_ENVIRONMENT = 'prd'
    ARTIFACTORY_ENV = 'prod'
    STAGE = 'prd'
}  else if (env.BRANCH_NAME.startsWith("release/") || (BRANCH_NAME == "qa")) {
    DEEP_ENVIRONMENT = 'qa'
    ARTIFACTORY_ENV = 'qa'
    STAGE = 'qa'
}   else if (BRANCH_NAME.startsWith("feature/") || (BRANCH_NAME == "dev")) {
    DEEP_ENVIRONMENT = 'dev'
    ARTIFACTORY_ENV = 'dev'
    STAGE = 'dev'
} else {
    error("This branch is not allowed")
    currentBuild.result = "ABORTED"
}

// Set S3 bucket name
S3_BUCKET_NAME = "${STREAM}-s3-library-${DEEP_ENVIRONMENT}"

// Set service account
SERVICE_ACCOUNT = "${DEEP_PRODUCT_NAME}-${STAGE}-aws"

// Information discovered on runtime
repoVersion = ''
lambdas = []

// -----------------------------------------------------------------------------
// Auxiliary functions
// -----------------------------------------------------------------------------

// Use AWS CLI to discover the current AWS Account ID
def discoverAccountID() {
    return sh(returnStdout: true, script: "aws sts get-caller-identity | jq -r \".Account\"").trim()
}

def initSetup() {
    sh(script: """
    apt update; apt install build-essential jq zip default-jdk -y --no-install-recommends
    make init
    """)
}

// def poetrySetup() {
//     sh(script: """
//     make install-poetry
//     make install
//     """)
// }

def uvSetup() {
    sh(script: """
    make install
    make sync
    """)
}

def formatAndLintCodebase() {
    sh(script: """
    make format
    make lint
    """)
}

def unitTest() {
    sh(script: "make test")
}

def buildWheelAndSyncArtifact() {
    sh(script: """
    make build
    echo 'Transferring wheel file to S3'
    aws s3 cp ./dist/ s3://${S3_BUCKET_NAME}/python/kumeza --recursive --exclude \"*.gz\"
    """)
}

// single quotes enclosure is necessary to solve dynamic variable assignment
def buildLambdaLayerZipAndSyncArtifact(package_semver) {
    sh(script: """
    make lambda-layer
    echo 'Transferring wheel file to S3 as a Lambda Layer zip'
    aws s3 cp out/kumeza.zip s3://${S3_BUCKET_NAME}/lambda-layers/kumeza-${package_semver}-none-any.zip
    aws s3 cp out/pyarrow.zip s3://${S3_BUCKET_NAME}/lambda-layers/pyarrow.zip
    """)
}

// def discoverPackageSemver() {
//     return sh(returnStdout: true, script: "poetry version | awk '{print \$2}'").trim()
// }

def discoverPackageSemver() {
    return sh(returnStdout: true, script: "uvx --from=toml-cli toml get --toml-path=pyproject.toml project.version").trim()
}

def copyArtifactsToS3Bucket() {
    sh(script: """
    echo 'Transferring wheel file to S3'
    aws s3 cp ./dist/ s3://${S3_BUCKET_NAME}/python/kumeza --recursive --exclude \"*.gz\"
    """)
}

def copyOdbcLibToS3Bucket() {
    sh(script: """
    echo 'Transferring ODBC shared library files to S3'
    aws s3 cp ./shared_lib/odbc s3://${S3_BUCKET_NAME}/odbc --recursive
    """)
}

// DEEP-Jenkins IAM role permission need to be modified
// by allowing lambda:PublishLayerVersion action
// !!Not valid for now!!
def deployLambdaLayer() {
    sh(script: """
    aws lambda publish-layer-version --layer-name ${ZIP_NAME} \
        --description "DAAS Common Library V2 SDK Layer" \
        --license-info "MIT" \
        --content S3Bucket=${S3_BUCKET_NAME},S3Key=python/kumeza.zip \
        --compatible-runtimes python3.9
    """)
}

pipeline {
    agent {
        kubernetes {
            inheritFrom "${SERVICE_ACCOUNT}"
            yaml """
                apiVersion: v1
                kind: Pod
                spec:
                    serviceAccountName: '${SERVICE_ACCOUNT}'
                    imagePullSecrets: [ 'artifactory-icloud-dev' ]
                    containers:
                    - name: awscli
                      image: art.pmideep.com/dockerhub/amazon/aws-cli:2.17.4
                      command:
                      - cat 
                      tty: true
                    - name: python
                      image: art.pmideep.com/dockerhub/python:3.10-bookworm
                      command:
                      - cat
                      tty: true
                      resources:
                        requests:
                            cpu: 1
                            memory: 1Gi
                        limits:
                            cpu: 2
                            memory: 2Gi
                    - name: sonarscanner
                      image: art.pmideep.com/dockerhub/sonarsource/sonar-scanner-cli:5
                      command:
                      - cat
                      tty: true
                    - name: zip
                      image: art.pmideep.com/dockerhub/javieraviles/zip:latest
                      command:
                      - cat
                      tty: true
            """
        }
    }
    stages {
        stage('Run CI?') {
            agent any
            steps {
                script {
                    if (sh(script: "git log -1 --pretty=%B | fgrep -ie '[skip ci]' -e '[ci skip]'", returnStatus: true) == 0) {
                        currentBuild.result = 'NOT BUILT'
                        error 'Aborting because commit message contains [skip ci]'
                    }
                }
            }
        }
        stage('Initial Setup') {
            steps {
                container('python') {
                    script {
                        echo "Using DEEP environment: ${DEEP_ENVIRONMENT}"
                        echo "Using Service Account: ${SERVICE_ACCOUNT}"
                        initSetup()
                        accountId = discoverAccountID()
                    }
                }
            }
        }
        // stage('Install and Setup Poetry') {
        //     steps {
        //         container("python") {
        //             script {
        //                 poetrySetup()
        //             }
        //         }
        //     }
        // }
        stage('Install and Setup uv package manager') {
            steps {
                container("python") {
                    script {
                        uvSetup()
                    }
                }
            }
        }
        stage('Format and Lint the Codebase') {
            steps {
                container("python") {
                    script {
                        formatAndLintCodebase()
                    }
                }
            }
        }
        stage('Unit Test') {
            steps {
                container("python") {
                    script {
                        unitTest()
                    }
                }
            }
        }
        stage('Build Wheel File and Sync Artifact') {
            steps {
                container("python") {
                    script {
                        buildWheelAndSyncArtifact()
                    }
                }
            }
        }
        stage('Build Lambda Layer Zip File and Sync Artifact') {
            steps {
                container("python") {
                    script {
                        package_semver = discoverPackageSemver()
                        buildLambdaLayerZipAndSyncArtifact(package_semver)
                    }
                }
            }
        }
        stage('Sync Shared Library File') {
            steps {
                container("python") {
                    script {
                        copyOdbcLibToS3Bucket()
                    }
                }
            }
        }
    }
}
{% endhighlight %}

### Conclusion
TBC