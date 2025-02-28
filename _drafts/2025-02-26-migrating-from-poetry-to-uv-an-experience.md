---
title:  "Migrating from Poetry to UV: An Experience"
seo_title: "migrating from poetry to uv: an experience"
seo_description: "migrating from poetry to uv: an experience"
date:   2025-02-26 00:00:00 +0700
categories:
  - Programming
tags:
  - Python
excerpt: "This post describes my experience migrating from Poetry to UV (+Ruff) as my Python packaging and build framework."
---
### Background
In recent times, Rust has been gaining some niche traction where it is being used as an underlying language for many Python libraries. Two of the most popular ones are [uv](https://astral.sh/blog/uv), which is touted as the fastest Python package installer and resolver and [Ruff](https://docs.astral.sh/ruff/?ref=blog.jerrycodes.com), which is touted as the fastest Python linter and code formatter. Both libraries are being developed and maintained by [Astral](https://astral.sh), which has the mission to make the Python ecosystem more productive.

### The Problem
I have been building and maintaining a lot of internal tools using Python with a similar vision to make the ecosystem in my workplace to be more productive while at the same time be more cost-efficient by reducing dependencies on external SaaS solution. I am using Poetry as my main Python dependency management and packaging and also as the backend to run linter tools such as `isort`, `Black`, `flake8` and unit tests.

With increasing amount of lines of code and dependencies, I noticed the CI/CD pipeline elapsed time is also growing and it means that the cost of running the pipeline is also increasing. The potential of using `uv` and `ruff` to reduce the time compels me to try them out. This post describes my experience of the migration and brief comparison of the final results between the two frameworks.

### The Steps
My CI/CD pipeline are defined in Jenkinsfile stages and it's triggered by BitBucket post webhook on BitBucket events. On high-level, the stages are:

1. BitBucket webhook triggered by pull request merge events.
2. Check whether CI run is needed.
3. Setup the build environment.
4. Install the Python package manager.
5. Format and lint the codebase.
6. Run unit testing.
7. Build and sync the artifacts to the runtime environment.

#### The `Makefile`
**Use the Same Poetry Steps**
{% highlight makefile %}
NAME := kumeza
UV := $(shell command -v uv 2> /dev/null)
RUFF := $(shell command -v ruff 2> /dev/null)

.DEFAULT_GOAL := help

.PHONY: help
help:
		@echo "Please use 'make <target> where <target> is one of:"
		@echo ""
		@echo " init				install prerequisites packages"
		@echo "	install				install uv package manager and ruff linter"
		@echo "	sync				install package dependencies"
		@echo "	clean				remove all temporary files"
		@echo "	lint				run the code linters"
		@echo "	format				format code according to formatter config"
		@echo "	test 				run all the tests"
		@echo " lambda-layer		build package as lambda layer"
		@echo ""
		@echo "Check the Makefile to know exactly what each target is doing."

.PHONY: init
init:
		python -m pip install --upgrade pip
		python -m pip install awscli
		python -m pip install git+https://github.com/awslabs/aws-glue-libs.git
		
.PHONY: install
install:
		@echo "Installing uv..."
		pip install uv
		@echo "Installing ruff..."
		pip install ruff
		@ruff --version

.PHONY: sync
sync:
		@if [ -z $(UV) ]; then echo "UV could not be found. See https://docs.astral.sh/uv/getting-started/installation/"; exit 2; fi
		$(UV) sync

.PHONY: clean
clean:
		find . -type d -name "__pycache__" | xargs rm -rf {};
		rm -rf .coverage .mypy_cache .pytest_cache ./dist ./htmlcov ./package

.PHONY: format
format: 
		$(UV) run isort --profile=black --lines-after-imports=2 ./tests/ $(NAME)
		$(UV) run black ./tests/ $(NAME)

.PHONY: lint
lint: 
		$(UV) run isort --profile=black --lines-after-imports=2 --check-only ./tests/ $(NAME)
		$(UV) run black --check ./tests/ $(NAME) --diff
		$(UV) run flake8 --ignore=W503,E501 ./tests/ $(NAME)
		$(UV) run mypy ./tests/ $(NAME) --ignore-missing-imports
		$(UV) run bandit -r $(NAME) -s B608

.PHONY: test
test: 
		$(UV) run pytest --cov $(NAME) --cov-report=term-missing --cov-report html --cov-fail-under 100 

build:
		$(UV) build

lambda-layer:
		$(UV) sync
		$(UV) build
		$(UV) run pip install --upgrade -t python dist/*.whl
		mkdir -p out; zip -r -q out/kumeza.zip python/* -i 'python/kumeza*' -x 'python/*.pyc'
		zip -r -q out/pyarrow.zip python/* -i 'python/*arrow*' -x 'python/*.pyc'
{% endhighlight %}

**Use Ruff Formatter Configuration**
{% highlight makefile %}
NAME := kumeza
UV := $(shell command -v uv 2> /dev/null)
RUFF := $(shell command -v ruff 2> /dev/null)

.DEFAULT_GOAL := help

.PHONY: help
help:
		@echo "Please use 'make <target> where <target> is one of:"
		@echo ""
		@echo " init				install prerequisites packages"
		@echo "	install				install uv package manager and ruff linter"
		@echo "	sync				install package dependencies"
		@echo "	clean				remove all temporary files"
		@echo "	lint				run the code linters"
		@echo "	format				format code according to formatter config"
		@echo "	test 				run all the tests"
		@echo " lambda-layer		build package as lambda layer"
		@echo ""
		@echo "Check the Makefile to know exactly what each target is doing."

.PHONY: init
init:
		python -m pip install --upgrade pip
		python -m pip install awscli
		python -m pip install git+https://github.com/awslabs/aws-glue-libs.git
		
.PHONY: install
install:
		@echo "Installing uv..."
		pip install uv
		@echo "Installing ruff..."
		pip install ruff
		@ruff --version

.PHONY: sync
sync:
		@if [ -z $(UV) ]; then echo "UV could not be found. See https://docs.astral.sh/uv/getting-started/installation/"; exit 2; fi
		$(UV) sync

.PHONY: clean
clean:
		find . -type d -name "__pycache__" | xargs rm -rf {};
		rm -rf .coverage .mypy_cache .pytest_cache ./dist ./htmlcov ./package

.PHONY: format
format: 
		$(RUFF) format --verbose

.PHONY: lint
lint: 
		$(RUFF) check

.PHONY: test
test: 
		$(UV) run pytest --cov $(NAME) --cov-report=term-missing --cov-report html --cov-fail-under 100 

build:
		$(UV) build

lambda-layer:
		$(UV) sync
		$(UV) build
		$(UV) run pip install --upgrade -t python dist/*.whl
		mkdir -p out; zip -r -q out/kumeza.zip python/* -i 'python/kumeza*' -x 'python/*.pyc'
		zip -r -q out/pyarrow.zip python/* -i 'python/*arrow*' -x 'python/*.pyc'
{% endhighlight %}

### Conclusion
TBC