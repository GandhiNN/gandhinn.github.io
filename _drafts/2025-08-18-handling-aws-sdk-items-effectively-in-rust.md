---
title:  "Handling AWS SDK Items Effectively in Rust"
seo_title: "handling aws sdk items effectively in rust"
seo_description: "handling aws sdk items effectively in rust"
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

Items (or structs) in AWS SDK for Rust usually has the same set of implementation method call with the attributes that it has (plus a dedicated `builder()` method, which is used if we want to build the item, but it's outside the scope of this post). It means that, if we use the above `Role` struct's attributes as the reference, this item will also have the following methods:

Notice that the struct `Role` has methods which take reference to itself to retrieve its attributes' references. This is useful, for example, if we are creating an external function to retrieve the values of roles that we have in our AWS tenant and use them for further processing downstream.

**Notice:** The RTFM Section Ends.
{: .notice}

# Minimize Allocation by Using the Implemented APIs to Retrieve the Attributes
To avoid extra heap allocation for the cloned `r` object, we can use the built-in methods already defined for the `aws_sdk_iam::Role` struct which takes a reference to itself (`&self`) as the first argument, which returns a reference containing the field's value. For example, the method `role_id(&self)` returns a `&str` containing the value of the `role_id` field.

Here is the updated code:

{% highlight rust %}
TBC
{% endhighlight %}

# Profiling memory allocation in Rust using `Heaptrack`
[Heaptrack](https://github.com/KDE/heaptrack/blob/master/README.md) is a heap memory profiler for Linux.

Heaptrack traces all memory allocations and annotates these events with stack traces. Some important metrics produced by it are memory footprints, memory leaks, memory allocation hotspots, and temporary allocations.

On Ubuntu (22.04) heaptrack can be installed via `apt`:

{% highlight bash %}
sudo apt install heaptrack heaptrack-gui
{% endhighlight %}

TBC

# Conclusion
TBC