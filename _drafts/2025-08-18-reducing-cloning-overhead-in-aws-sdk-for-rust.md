---
title:  "Reducing Cloning Overhead in AWS SDK for Rust"
seo_title: "Reducing Cloning Overhead in AWS SDK for Rust"
seo_description: "Reducing Cloning Overhead in AWS SDK for Rust"
date:   2025-08-18 00:00:00 +0700
categories:
  - Programming
tags:
  - Rust
excerpt: "This post explores how to handle AWS SDK for Rust items efficiently by minimizing unnecessary cloning."
toc: true
toc_label: "Table of Contents"
---
# Overview
I was building an audit tool designed to programmatically retrieve key attributes related to IAM access keys. This includes identifying the users associated with each key pair, determining whether the keys are active or inactive, and capturing metadata such as creation dates and last used timestamps. The goal was to streamline visibility into access key usage across the environment.  

# Using `clone()` Everywhere 
I began by defining a custom type to capture the necessary information:

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

Next, I iterated through the paginated stream returned by the `list_roles()` API from `aws_sdk_iam::Client`.

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

The error message indicates that `r` is of type `&aws_sdk_iam::types::Role`, meaning it's a shared reference to a `Role` struct. When I first access the path field using the dot (.) operator, Rust automatically dereferences `r` to retrieve the value. However, on subsequent access, such as retrieving `role_id`, the compiler throws an error because the original reference has already been dereferenced, and Rust's borrowing rules prevent reusing it in that context. 

One of the compiler's suggestions to resolve the issue is to clone the value of `r`. While this might seem like a quick fix, it's easy to fall into the trap of doing something like the following, without fully considering the impact on memory usage and performance:

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

With these changes, the code now compiles successfully. However, it introduces additional memory allocations, as we're cloning the entire `r` object each time we access one of its attributes to populate our custom `Role` struct. While functionally correct, this approach can be inefficient—especially in scenarios where performance and memory usage are critical.

Rust’s powerful RAII (Resource Acquisition Is Initialization) paradigm offers robust memory management capabilities. However, since RAM remains a finite resource, it's crucial to stay vigilant and intentional about how memory is allocated and used, especially in performance-critical applications. Understanding and optimizing allocation patterns can make a significant difference in overall efficiency. 

Frequent use of the `clone()` operation is often a red flag when it comes to efficient memory management. For instance, if the `r` object is 50 KB in size, cloning it effectively doubles the memory footprint to 100 KB at runtime. In a concurrent environment handling 1,000 `r` objects simultaneously, this could result in an additional 100 MB of memory allocation per second—quickly adding up and potentially impacting system performance. 

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

Heaptrack monitors all memory allocations and enriches them with stack traces, providing key insights such as memory footprint, leaks, allocation hot spots, and transient allocations.

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

|--------------+-----------------|
| With Cloning | Without Cloning |  
|:------------:|:---------------:|  
|![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2025-08-18-reducing-cloning-overhead-in-aws-sdk-for-rust/heaptrack_with_cloning-summary-20250815.png){: .align-center}|![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2025-08-18-reducing-cloning-overhead-in-aws-sdk-for-rust/heaptrack_with_cloning-20250815.png){: .align-center}|
|![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2025-08-18-reducing-cloning-overhead-in-aws-sdk-for-rust/heaptrack_without_cloning-summary-20250815.png){: .align-center}|![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2025-08-18-reducing-cloning-overhead-in-aws-sdk-for-rust/heaptrack_without_cloning-20250815.png){: .align-center}|

# Conclusion
As we've seen, managing memory allocations in Rust, especially when working with the AWS SDK, isn't as daunting as it might seem. Many SDK types offer convenient accessor methods that let us retrieve values without cloning entire objects, making our code both cleaner and more efficient. These techniques aren't limited to IAM; they apply equally well to other AWS services like Glue and S3. And when deeper analysis is needed, tools like Heaptrack provide valuable insights into runtime behavior. In my case, it clearly showed how excessive use of `clone()` can lead to significant memory overhead. And that's it for now! Stay tuned for my future posts in the future!
