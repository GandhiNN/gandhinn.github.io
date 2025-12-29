---
title:  "Simple Housekeeping Script for Rust Projects"
seo_title: "simple housekeeping script for rust projects"
seo_description: "automating cleanup and maintenance tasks for rust projects using shell scripts"
date:   2025-12-29 00:00:00 +0700
categories:
  - Programming
tags:
  - Rust
  - Shell
  - Automation
  - DevOps
  - CLI
excerpt: "This post describes how to create a simple housekeeping script to automate cleanup and maintenance tasks for Rust projects..."
toc: true
toc_label: "Table of Contents"
---

## Overview

If you are like me, you might have the following directory tree under your $HOME:

{% highlight bash %}
$HOME/
├── Documents/
├── Downloads/
├── study/
│   ├── rust/
│   │   ├── ownership-basics/
│   │   ├── error-handling/
│   │   ├── async-programming/
│   │   ├── web-frameworks/
│   │   ├── cli-tools/
│   │   └── memory-management/
│   ├── python/
│   │   ├── data-structures/
│   │   ├── web-scraping/
│   │   └── machine-learning/
│   ├── go/
│   │   ├── concurrency/
│   │   ├── microservices/
│   │   └── testing-patterns/
│   ├── javascript/
│   │   ├── react-hooks/
│   │   ├── node-apis/
│   │   └── typescript-basics/
│   └── cpp/
│       ├── templates/
│       ├── smart-pointers/
│       └── algorithms/
└── projects/
{% endhighlight %}

As you can see, every concept I learn gets its own directory under the programming language it's written in. This works great as a museum of my learning journey, except when these educational breadcrumbs turn into a storage nightmare. This post shows how I created a simple housekeeping bash script to reclaim that precious storage space.  

## A Brief Explanation of the Challenge

Rust produces build artifacts that can consume significant storage space for the following reasons:

1. Larger binaries than C/C++ equivalents due to static linking by default (though this is configurable)  
2. Unoptimized debug builds that retain additional metadata and debug symbols
3. Deep dependency trees: `cargo` compiles all project dependencies recursively. For example, `tokio` is a large crate with extensive dependencies, resulting in build artifacts that can reach hundreds of megabytes to several gigabytes  

To reclaim disk space, we can use the `cargo clean` command, which removes artifacts from the target directory that Cargo has generated. By default `cargo clean` deletes the entire `target/` directory. However, if you have a similar setup with mine (I hope not!), manually running `cargo clean` in every directory becomes impractical and time-consuming.

## Solution

TBC

## Conclusion

TBC