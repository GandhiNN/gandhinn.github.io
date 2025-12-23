---
title:  "Building an Amazon Athena Shell using Rust"
seo_title: "building an amazon athena shell using rust"
seo_description: "building an interactive command-line shell for amazon athena using rust programming language"
date:   2025-12-25 00:00:00 +0700
categories:
  - Programming
tags:
  - AWS
  - Rust
  - Athena
  - CLI
excerpt: "This post describes how to build an interactive command-line shell for Amazon Athena using Rust programming language..."
toc: true
toc_label: "Table of Contents"
---

## Overview

Amazon Athena is an interactive query service to analyze data in Amazon S3 using SQL-like syntax. I have been using this service on regular basis to check, analyze, and sometimes to pre-process data sitting in my data lake for business consumption. As a CLI person, I found that traversing AWS console to open the query editor is one too many steps for my liking. This post demonstrates my experience building a mini-shell where I can execute the same query directly from my Linux shell.

## Prerequisites

- Rust toolchain (1.70+) installed via [rustup](https://rustup.rs/)
- Valid AWS credentials
- An existing Athena database and S3 bucket for query results

## Project Setup

We'll use the following project structure:
{% highlight bash %}

athena-shell
├── Cargo.lock
├── Cargo.toml
├── src
│   ├── client.rs
│   ├── error.rs
│   ├── main.rs
│   └── repl.rs
└── target

{% endhighlight %}

First, let's set our crate dependencies in Cargo.toml:

{% highlight yaml %}

[dependencies]
anyhow = "1.0"
aws-config = "1.8"
aws-runtime = "1.5"
aws-sdk-athena = "1.97"
aws-smithy-types = "1.3.5"
aws-types = "1.3.11"
directories = "6.0"
thiserror = "2.0"
tokio = { version = "1.48", features = ["full"] }

{% endhighlight %}

TBC

## Implementation

### Creating AWS SDK Config Builder


## Usage Examples

TBC

## Conclusion

TBC
