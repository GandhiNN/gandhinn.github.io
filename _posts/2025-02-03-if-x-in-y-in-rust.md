---
title:  "Search if an Item Exists Inside a Collection in Rust"
seo_title: "search if an item exists inside a collection in rust"
seo_description: "search if an item exists inside a collection in rust"
date:   2025-02-03 00:00:00 +0700
categories:
  - Programming
tags:
  - Rust
  - Python
---
In Python, you can  use the `in` keyword to check if a value is present in a sequence (list, range, string, etc):

{% highlight python %}
# a list of car makers
carmakers = ["Toyota", "Mercedes Benz", "BMW", "Honda"]

# check if BYD in list or not
if "BYD" in carmakers:
    print("BYD is in car makers")
{% endhighlight %}

We can do the same in Rust by using the `if let` statement to allows matching on various options on a variable.

For the example, let's write a function in Rust to count the number of vowels (`a`, `i`, `u`, `e`, `o`) in a 
given word or sentence, regardless of case (i.e. `A` is the same with `a`):

{% highlight rust %}
pub fn count_vowels(strinput: String) -> i32 {
    let mut num_vowels: i32 = 0;
    for char in strinput.to_ascii_lowercase().chars() {
        if let 'a' | 'i' | 'u' | 'e' | 'o' = char {
            num_vowels += 1;
        }
    }
    num_vowels
}
{% endhighlight %}

Now, let's define our test case:

{% highlight rust %}
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_count_vowels_in_a_sentence() {
        let sentence = String::from("Who are you?");
        let expected = 5;
        let result = count_vowels(sentence);
        assert_eq!(result, expected);
    }

    #[test]
    fn test_count_vowels_in_a_word() {
        let word = String::from("ASDsdaf!@3%sdsga#5shdsauyetow");
        let expected = 7;
        let result = count_vowels(word);
        assert_eq!(result, expected);
    }
}
{% endhighlight %}

Now let's run the test:

{% highlight bash %}
$ cargo test -q

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s


running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
{% endhighlight %}

You can check out the full code in this [GitHub repo][github-repo] for more info.

[github-repo]: https://github.com/GandhiNN/rust-leetcode/blob/master/problem_set/easy/count_vowels/src/lib.rs
