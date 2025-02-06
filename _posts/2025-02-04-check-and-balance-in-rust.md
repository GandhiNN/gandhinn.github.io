---
title:  "Check if a list is balanced in Rust"
seo_title: "Check if a list is balanced in Rust"
seo_description: "Check if a list is balanced in Rust"
date:   2025-02-04 00:00:00 +0700
categories:
  - Programming
tags:
  - Rust
---
# Can you balance?
> Given a non-empty list of integers, return `true` if there is a place to split the list so that the sum of the numbers on one side is equal to the sum of the numbers on the other side. 

For example, given the vector of [1,5,3,3], the function would return `true` because we can split it in such a way that the sum of `[1, 5]` is the same with the sum of `[3, 3]`. 

Another example is a vector of [7,3,4] that would also return `true` because the sum of [7] is the same with the sum of `[3, 4]`. 

Only the boolean value needs to be returned, not the split index point.