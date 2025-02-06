---
title:  "Check Whether a List is Balanced in Rust"
seo_title: "check whether a list is balanced in rust"
seo_description: "check whether a list is balanced in rust"
date:   2025-02-04 00:00:00 +0700
categories:
  - Programming
tags:
  - Rust
---
This post is about a port of solution of an interesting brain teaser that I've solved a couple years ago in Rust. 

> Given a non-empty list of integers, return `true` if there is a place to split the list so that the sum of the numbers on one side is equal to the sum of the numbers on the other side. 

For example, given the vector of `[1,5,3,3]`, the function would return `true` because we can split it in such a way that the sum of `[1, 5]` is the same with the sum of `[3, 3]`. 

Another example is a vector of `[7,3,4]` that would also return `true` because the sum of `[7]` is the same with the sum of `[3, 4]`. 

The function shall return `false` when given a vector of `[1, 2, 2]`.

Only the boolean value needs to be returned, not the split index point.

In Rust, there's an iterator method called `split_off()` that we can use for this purpose:

{% highlight rust %}
pub fn split_off(&mut self, at: usize) -> Vec<T>
{% endhighlight %}

From the signature, we see this method splits the collection into two at the given index. This method returns a newly allocated vector which contains elements `[0, at)`, while the original vector will be mutated into a collection which contains elements `[at, len)`. 

Note that the original vector is consumed and cannot be used anymore.

{% highlight rust %}
pub fn check_and_balance(input: Vec<i32>) -> bool {
    for i in 1..input.len() - 1 {
        let mut vec = input.clone();
        let vec2: Vec<i32> = vec.split_off(i);
        if vec2.iter().sum::<i32>() == vec.iter().sum::<i32>() {
            return true;
        } else {
            continue;
        }
    }
    false
}
{% endhighlight %}

Here, we start iterating and splitting off the input vector and do a sum comparison between the resulted two vectors. 

As mentioned earlier, because the original input vector cannot be re-used, we have to clone the input vector on every iteration. We are returning early from the loop as soon as we found the first occurence where the sum of the element of the left vector equals to the sum of the element of the right one. 

In worst-case scenario, we will iterate `n = (input.len() - 1)` times.

Let's define the unit test:

{% highlight rust %}
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_check_and_balance() {
        let input = vec![1, 5, 3, 3];
        let expected = true;
        assert_eq!(check_and_balance(input), expected);
    }

    #[test]
    fn test_check_and_balance_v2() {
        let input = vec![7, 3, 4];
        let expected = true;
        assert_eq!(check_and_balance(input), expected);
    }

    #[test]
    fn test_check_and_balance_v3() {
        let input = vec![1, 2, 2];
        let expected = false;
        assert_eq!(check_and_balance(input), expected);
    }
}
{% endhighlight %}

The test results:
{% highlight bash %}
$ cargo test -q

running 3 tests
...
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s


running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
{% endhighlight %}
