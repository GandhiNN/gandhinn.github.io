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

I created the following alias in my `$HOME/.bash_profile` to automate cleanup across multiple Rust projects. What it does basically are:  

1. Iterates through directories
2. Identifies valid Rust projects by checking for `Cargo.toml`
3. Runs `cargo clean` and check for the number of deleted files and bytes
4. Outputs a pipe-separated report of the cleanup results

Just remember to run the command in your parent directory i.e. `$HOME/study/rust`  

{% highlight bash %}
cargo_purge()
{
        echo "dir|status|removedFiles|size|errMessage"
        for dir in "$(ls)"
        do
                cd "${dir}" # handle directories with whitespaces
                ls "Cargo.toml" > /dev/null 2>&1
                lsRetVal=$?

                # Initialize variables
                removedFiles=0
                size="0"
                errMessage="NA"

                if [ $lsRetVal -ne 0 ]; then
                        status="ERROR"
                        errMessage="not a cargo project"
                else
                        out=$(cargo clean 2>&1) # redirect STDERR TO STDOUT stream to capture `cargo clean` output
                        status=$(echo ${out} | awk -F ' ' '{print $1}')

                        if [ $status == "Removed" ]; then
                                status="SUCCESS"
                                removedFiles=$(echo ${out} | awk -F ' ' '{print $2}')
                                if [ $removedFiles != "0" ]; then
                                        size=$(echo ${out} | awk -F ' ' '{print $4}')
                                fi
                        else
                                status="ERROR"
                                errMessage=$(echo ${out} | awk -F ':' '{print $2}' | xargs) # use xargs to trim whitespace from error messages
                        fi
                fi

                echo "${dir}|${status}|${removedFiles}|${size}|${errMessage}"
                cd ..
        done
}
{% endhighlight %}

Sample output:

{% highlight bash %}
dir|status|removedFiles|size|errMessage
ownership-basics|SUCCESS|245|15.2MB|NA
error-handling|SUCCESS|0|0|NA
async-programming|ERROR|0|0|not a cargo project
web-frameworks|SUCCESS|1,234|156.7MB|NA
{% endhighlight %}

## Conclusion

Developing this simple bash function has been a cool side learning quest for me.

I think this concept can also be applied to other languages and build systems as well. For example, in Python projects for `.pyc` and `__pycache__` cleanup.  

Last words, I hope this simple script can be beneficial in your own Rust development workflow. ~~I think this idea could be expanded into a Cargo sub-command as well (`cargo clean-recursive`, anyone?)~~ Apparently the `cargo clean-recursive` subcommand to clean all projects under specified directory already exists: [cargo-clean-recursive](https://crates.io/crates/cargo-clean-recursive), so please use that one instead.  
  
With that said, you know it's always a good feeling to have a spacious storage and fast builds!  
