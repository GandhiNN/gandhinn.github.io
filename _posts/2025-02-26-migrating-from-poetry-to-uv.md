---
title:  "Migrating from Poetry to UV"
seo_title: "migrating from poetry to uv"
seo_description: "migrating from poetry to uv"
date:   2025-02-26 00:00:00 +0700
categories:
  - Programming
tags:
  - Python
excerpt: "This post describes my experience migrating from Poetry to UV (+Ruff) as my Python packaging and build framework."
toc: true
toc_label: "Table of Contents"
---
### Background
Lately, Rust has been gaining some traction as an underlying language for many Python libraries. Two of the most popular ones are [uv](https://astral.sh/blog/uv) and [Ruff](https://docs.astral.sh/ruff/?ref=blog.jerrycodes.com), which are touted as the fastest Python package manager, linter and code formatter. Both libraries are being developed and maintained by [Astral](https://astral.sh), a company who aims to make the Python ecosystem more productive.

### The Problem
I've been building and maintaining some internal SDKs using Poetry as my main Python package dependency management. It's also used as the backend to run linter tools such as `isort`, `Black`, `flake8` and run unit tests. With the increasing amount of lines of code and dependencies, I noticed the runtime duration of my Jenkins pipeline is also growing, which implies that the cost of running the CI/CD pipeline is also increasing. 

`uv` and `ruff`, which are written in Rust, promises some potential to reduce the time needed to build my SDK, which compels me to try them out. This post describes my experience of the migration and brief comparison of the final results between the two frameworks.  

### Preparations

Pip can be used to install `uv` and `ruff`:

{% highlight bash %}
pip install uv
pip install ruff
{% endhighlight %}

To migrate from Poetry to uv, we can use the migration tool by running the following command in your project root:

{% highlight bash %}
uvx migrate-to-uv
{% endhighlight %}

### The Build Pipeline
My CI/CD pipeline are defined in Jenkinsfile stages and it's triggered by BitBucket post webhook on BitBucket events. On high-level, the stages are:

1. BitBucket webhook triggered by pull request merge events.
2. Check whether CI run is needed.
3. Setup the build environment.
4. Install the Python package manager.
5. Format and lint the codebase.
6. Run unit testing.
7. Build and sync the artifacts to the runtime environment.

Steps (3) through (6) are defined in a Makefile, which will be called during the Jenkins build itself.

#### Naive Migration from Poetry to Uv
At first, I was just migrating naively from Poetry to uv, i.e., I just replaced all `poetry` commands to `uv` with the same linter configuration:

{% highlight makefile %}

/* Note: Only relevant blocks are shown for brevity */

NAME := kumeza
UV := $(shell command -v uv 2> /dev/null)
RUFF := $(shell command -v ruff 2> /dev/null)

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

.PHONY: format
format: 
		$(UV) run isort --profile=black --lines-after-imports=2 ./tests/ $(NAME)
		$(UV) run black ./tests/ $(NAME)

.PHONY: lint
lint: 
		$(UV) run flake8 --ignore=W503,E501 ./tests/ $(NAME)
		$(UV) run mypy ./tests/ $(NAME) --ignore-missing-imports
		$(UV) run bandit -r $(NAME) -s B608

.PHONY: test
test: 
		$(UV) run pytest --cov $(NAME) --cov-report=term-missing --cov-report html --cov-fail-under 100 

build:
		$(UV) build

{% endhighlight %}

The result is Uv performed slightly better than Poetry:

|---------+--------|
| Builder | Result |  
|:-------:|:------:|  
|Poetry|![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2025-02-26-migrating-from-poetry-to-uv/poetry-build_steps-20250227.png){: .align-center}|
|Uv|![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2025-02-26-migrating-from-poetry-to-uv/uv-build-steps-same_with_poetry-20250227.png){: .align-center}|

#### Using Ruff Default Formatter Configuration
Ruff's formatter is designed as a drop-in replacement for `Black`, but with more focus on performance and compatibility, due to the widespread use of Black in the Python ecosystem. Thus, I think that Ruff's default configuration will be good enough to ensure proper formatting and readability, and also PEP compliance of my codebase, without having to turn too much knobs and levers.

The following makefile describes the adjusted steps:

{% highlight makefile %}

/* Note: Only relevant blocks are shown for brevity */

NAME := kumeza
UV := $(shell command -v uv 2> /dev/null)
RUFF := $(shell command -v ruff 2> /dev/null)

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

{% endhighlight %}

This time Uv performed much better than Poetry, which as expected, is down to the performance of the default Ruff formatter configuration:

|---------+--------|
| Builder | Result |  
|:-------:|:------:|  
|Poetry|![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2025-02-26-migrating-from-poetry-to-uv/poetry-build_steps-20250227.png){: .align-center}|
|Uv|![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2025-02-26-migrating-from-poetry-to-uv/uv-build_steps-20250227.png){: .align-center}|

### Conclusion
The migration from Poetry to Uv in my work has been a pleasant and rewarding experience. Even though uv does not have feature parity with Poetry yet, but it does a great job of utilizing similar API to provide a near-seamless migration. Furthermore, it does provide some improvements in my development chain. The numbers may seem small, but if you multiply it by the number of build pipeline that runs every day, it could accumulate into a quite significant platform cost savings.

