---
title:  "Effective Memory Allocation: A Case Study Using AWS SDK for Rust"
seo_title: "effective memory allocation: a case study using aws sdk for rust"
seo_description: "effective memory allocation: a case study using aws sdk for rust"
date:   2025-08-18 00:00:00 +0700
categories:
  - Programming
tags:
  - Rust
excerpt: "This post will describe how to handle AWS SDK items efficiently in AWS SDK for Rust by avoiding unnecessary object cloning."
toc: true
toc_label: "Table of Contents"
---
# Overview
I was developing an audit tool to programmatically retrieve all the IAM access key roles attributes e.g. users owning the key pairs, active/inactive status of the keys, creation date, and last used date.   

# Using `clone()` Everywhere 
Let's say that we want to retrieve all IAM users configured in our AWS tenant. We will start by defining a new type to store the information:

{% highlight rust %}
use serde::Serialize;
use tabled::Tabled;

#[derive(Tabled, Debug, Serialize)]
pub struct Role {
    pub path: String,
    pub id: String,
    pub arn: String,
    pub create_date: String,
    pub max_session_duration: i32,
    pub role_last_used: String,
}

let mut roles: Vec<Role> = Vec::new();
{% endhighlight %}

The next thing we do is to iterate through the paginated stream of the `aws_sdk_iam::Client`'s `list_roles()` API:

{% highlight rust %}

use aws_sdk_iam::Client;

let client = Client::new(&config);

let mut stream = client.list_roles().into_paginator().send();
while let Some(output) = stream.next().await {
    match output {
        Ok(x) => {
            roles = x
                .roles()
                .into_iter()
                .map(|r| Role {
                    path: r.path,
                    id: r.role_id, // cannot move out from shared reference
                    arn: r.arn,
                    create_date: r.create_date.to_string(),
                    max_session_duration: r.max_session_duration.unwrap_or(0),
                    role_last_used: {
                        match r.role_last_used() {
                            Some(last_used) => match last_used.last_used_date {
                                Some(x) => x.to_string(),
                                None => String::from("1970-01-01T00:00:00Z"),
                            },
                            None => String::from("1970-01-01T00:00:00Z"),
                        }
                    },
                })
                .collect::<Vec<Role>>();
        }
        Err(e) => eprintln!("{:#?}", e),
    }
}
println!("{:#?}", roles);
{% endhighlight %}

The Rust compiler will throw an error:

{% highlight bash %}
error[E0382]: use of moved value: `r`
{% endhighlight %}

The error message means us that `r` has the type of `&aws_sdk_iam::types::Role` i.e. it's a shared reference to `Role` struct. The compiler throws us an error because when the first time we are accessing the field `path` by using `.` operator, it will automatically dereference the `Role` as necessary. Hence, the next time we try to access the field `role_id`, the original `r` object no longer points to anything as it has been dereferenced before. 

One of the suggestions coming from the compiler to work around the issue is to clone the value of `r`. Hence, we might be tempted to do like the following:

{% highlight bash %}
// ...
            roles = x
                .roles()
                .into_iter()
                .map(|r| Role {
                    path: r.clone().path,
                    id: r.clone().role_id, 
                    arn: r.clone().arn,
                    create_date: r.clone().create_date.to_string(),
                    max_session_duration: r.clone().max_session_duration.unwrap_or(0),
                    role_last_used: {
                        match r.role_last_used() {
                            Some(last_used) => match last_used.last_used_date {
                                Some(x) => x.to_string(),
                                None => String::from("1970-01-01T00:00:00Z"),
                            },
                            None => String::from("1970-01-01T00:00:00Z"),
                        }
                    },
                })
                .collect::<Vec<Role>>();
// ...
{% endhighlight %}

Now the code will compile. However, it comes with some extra memory allocations because we are cloning the whole `r` object everytime we retrieve its attribute and assign it into our custom `Role` struct.

Even though Rust is a powerful programming language with its RAII paradigm, given that RAM is still a limited resource, it is important for us to be aware and prudent on what is going on with the memory, especially the allocation processes. 

The many usages of `clone()` operation is not a good sign that we are. If the size of `r` object is 50 KB, cloning will double that to 100 KB during runtime. In a concurrent environment, if the code is handling 1000 `r` object at a same time, that will be an extra allocation of 100 MB per second. 

**Notice:** The RTFM Section Starts.
{: .notice}

## Common type definition of AWS SDK for Rust
Most of the structs that you will see in AWS SDK for Rust usually will have the following type definition. For example, the following snippet is the definition `Role` struct in the `aws_sdk_iam` crate:

{% highlight rust %}
#[non_exhaustive]
pub struct Role {
    pub path: String,
    pub role_name: String,
    pub role_id: String,
    pub arn: String,
    pub create_date: DateTime,
    pub assume_role_policy_document: Option<String>,
    pub description: Option<String>,
    pub max_session_duration: Option<i32>,
    pub permissions_boundary: Option<AttachedPermissionsBoundary>,
    pub tags: Option<Vec<Tag>>,
    pub role_last_used: Option<RoleLastUsed>,
}

impl Role {
    pub fn path(&self) -> &str,
    pub fn role_name(&self) -> &str,
    pub fn role_id(&self) -> &str,
    pub fn arn(&self) -> &str,
    pub fn create_date(&self) -> &DateTime,
    pub fn assume_role_policy_document(&self) -> Option<&str>,
    pub fn description(&self) -> Option<&str>,
    pub fn max_session_duration(&self) -> Option<i32>,
    pub fn permissions_boundary(&self) -> Option<&AttachedPermissionsBoundary>,
    pub fn tags(&self) -> &[Tag],
    pub fn role_last_used(&self) -> Option<&RoleLastUsed>,
    pub fn builder() -> RoleBuilder
}

{% endhighlight %}

## What is `impl` in Rust?
As per Rust official documentation, `impl` is "a keyword which defines implementations of functionality for a type, or a type implementing some functionality."

There are two uses of the `impl` keyword:
1. An `impl Trait` which can be used to designate a type that implements a trait called `Trait`.
2. An `impl` block which is used to implement some functionality for a type => **This is what we'll be focusing on this post**.

In Rust, an implementation consists of definitions of functions and constants. A function defined in an `impl` block can be:

1. Standalone, meaning it does not take the struct as the argument, and will result in a new and standalone object. One example of this is `Vec::new()` which is used to create a new vector object.

2. A method-call, meaning the function takes the struct (either `self`, `&self`, or `&mut self`) as its first argument to return some value. One example of this is `vec.len()` method.

In the AWS SDK for Rust, most items or structs typically come with a consistent set of method implementations that mirror their attributes. These include accessor methods and a dedicated `builder()` function, handy for creating new instances, though that's beyond the scope of this discussion. Taking the `Role` struct as an example, you can expect it to provide methods corresponding to its fields, such as `role_id()`, which allows you to retrieve the value of the `role_id` attribute directly.

**Notice:** The RTFM Section Ends.
{: .notice}

# Minimize Allocation by Using the Implemented APIs to Retrieve the Attributes
To avoid unnecessary heap allocations when cloning the r object, it's more efficient to utilize the built-in methods provided by the `aws_sdk_iam::Role` struct. These methods accept a reference to the struct (`&self`) and return a reference to the desired field, eliminating the need for cloning. For instance, calling `role_id(&self)` returns a `&str` that directly references the value of the `role_id` field, streamlining memory usage without sacrificing readability.

Here is the updated code:

{% highlight bash %}
// ...
            roles = x
                .roles()
                .into_iter()
                .map(|r| Role {
                    path: r.path().to_owned(),
                    id: r.role_id().to_owned(), 
                    arn: r.arn().to_owned(),
                    create_date: r.create_date().to_string(),
                    max_session_duration: r.max_session_duration.unwrap_or(0),
                    role_last_used: {
                        match r.role_last_used() {
                            Some(last_used) => match last_used.last_used_date {
                                Some(x) => x.to_string(),
                                None => String::from("1970-01-01T00:00:00Z"),
                            },
                            None => String::from("1970-01-01T00:00:00Z"),
                        }
                    },
                })
                .collect::<Vec<Role>>();
// ...
{% endhighlight %}

While the updated code still compiles successfully and produces the same output as before, the real question is: does it actually reduce memory allocations compared to the previous version? To validate this improvement, we need to go beyond functional correctness and examine the runtime behavior more closely.

# Profiling memory allocation in Rust using `Heaptrack`
[Heaptrack](https://github.com/KDE/heaptrack/blob/master/README.md) is a heap memory profiler for Linux.

Heaptrack traces all memory allocations and annotates these events with stack traces. Some important metrics produced by it are memory footprints, memory leaks, memory allocation hotspots, and temporary allocations.

Heaptrack is split into two parts:

1. The data collector, i.e. the `heaptrack` binary itself.
2. The analyzer GUI called `heaptrack_gui` (needs Qt5 and KF5 dependencies; make sure both are available in your OS) 

On Ubuntu (22.04) Heaptrack can be installed via `apt`:

{% highlight bash %}
sudo apt install heaptrack heaptrack-gui
{% endhighlight %}

Heaptrack supports Rust binaries natively, so it's important to run it directly on your compiled application, not via heaptrack cargo run, which would end up profiling Cargo itself rather than your actual code. Additionally, ensure your Rust binaries include the necessary debug symbols. If you're building in release mode, double-check that debug symbols are enabled in your Cargo.toml to get meaningful profiling results.:

{% highlight toml %}
[profile.release]
debug = true
{% endhighlight toml %}

An example usage for the above code looks like this:

{% highlight bash %}
heaptrack target/release/iam_role_with_cloning
heaptrack target/release/iam_role_without_cloning
{% endhighlight %}

Afterwards, heaptrack writes gzipped result files for each profiled binary and then we can inspect and visualize the results into graphs using `heaptrack_gui`. 

An example usage looks like this:

{% highlight bash %}
heaptrack_gui heaptrack.cargo.10397.zst
heaptrack_gui heaptrack.cargo.8423.zst
{% endhighlight %}

In my analysis, I focused on the "Summary" and "Allocations" tabs to monitor memory usage trends over time. One key insight was that leveraging clone() significantly impacts memory behavior—specifically, it nearly doubles the number of allocations during runtime. This observation highlights the importance of being mindful about cloning operations, especially in performance-sensitive contexts.

# Conclusion
TBC