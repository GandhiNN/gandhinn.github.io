---
title:  "Destructuring the Returned Struct in AWS SDK for Rust"
seo_title: "destructuring the returned struct in aws sdk for rust"
seo_description: "destructuring the returned struct in aws sdk for rust"
date:   2025-08-15 00:00:00 +0700
categories:
  - Programming
tags:
  - Rust
excerpt: "This post will describe how to destructure the returned struct from AWS SDK for Rust while avoiding cloning the whole object."
toc: true
toc_label: "Table of Contents"
---
# Overview
TBC

# Common type definition of AWS SDK for Rust
I'll be using `aws_sdk_iam` as the reference in this post.

The following structure is returned as a response element in several API operations that interact with IAM roles. You'll also find the similar structure in many crates of AWS SDK for Rust:

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
{% endhighlight %}

## What is `impl` in Rust?
As per Rust official documentation, `impl` is "a keyword which defines implementations of functionality for a type, or a type implementing some functionality."

There are two uses of the `impl` keyword:
1. An `impl Trait` which can be used to designate a type that implements a trait called `Trait`.
2. An `impl` block which is used to implement some functionality for a type => **This is what we'll be focusing on this post**.

An implementation consists of definitions of functions and constants. A function defined in an `impl` block can be:
1. Standalone, meaning it does not take the struct as the argument, and will result in a new and standalone object. One example of this is `Vec::new()` which is used to create a new vector object.
2. A method-call, meaning the function takes the struct (either `self`, `&self`, or `&mut self`) as its first argument. One example of this is `vec.len()` method.

Items (or structs) in AWS SDK for Rust usually has the same set of implementation method call with the attributes that it has (plus a dedicated `builder()` method, which is used if we want to build the item, but it's outside the scope of this post). It means that, if we use the above `Role` struct's attributes as the reference, this item will also have the following methods:

{% highlight rust %}
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

## Using the struct's attributes or the implemented APIs?
Notice that the struct `Role` has methods which take reference to itself to retrieve its attributes' references. This is useful, for example, if we are creating an external function to retrieve the values of roles that we have in our AWS tenant and use them for further processing downstream.

# Using it in practice
TBC

# Conclusion
TBC