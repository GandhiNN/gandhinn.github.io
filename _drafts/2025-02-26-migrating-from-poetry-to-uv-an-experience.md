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

.PHONY: install-uv
install-uv:
		@echo "Installing uv..."
		pip install uv
		@uv --version

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
		$(UV) install --only main --sync
		$(UV) build
		$(UV) run pip install --upgrade -t python dist/*.whl
		mkdir -p out; zip -r -q out/kumeza.zip python/* -i 'python/kumeza*' -x 'python/*.pyc'
		zip -r -q out/pyarrow.zip python/* -i 'python/*arrow*' -x 'python/*.pyc'
{% endhighlight %}

### Conclusion
TBC