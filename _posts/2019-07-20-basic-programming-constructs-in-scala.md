---
title:  "Basic Programming Constructs in Scala"
seo_title: "basic programming constructs in scala"
seo_description: "basic programming constructs in scala"
date:   2019-07-20 00:00:00 +0700
categories:
  - Programming
tags:
  - Scala
excerpt: "I am in the middle of Jose Portilla’s highly entertaining Scala and Spark for Big Data and Machine Learning course in Udemy. ....."
---
### Overview
I am in the middle of [Jose Portilla’s highly entertaining Scala and Spark for Big Data and Machine Learning course in Udemy](https://www.udemy.com/course/scala-and-spark-for-big-data-and-machine-learning/). The course delivery is on point and easy to follow for novices like me; I could not recommend this course more highly.

This note is my (naive) answers to Section 8’s Scala Programming Exercise on functions and flow control, and solely written for posterity (and to sharpen up my writing skills). Of course the codes will feel unoptimized to the pros, so feel free if you would like to suggest improvement to my codes in the comment section at the end of this post!

Note: I rarely code to solve problems not related to my daily work in telco domain, so it was fun knowing that I could answer this kind of questions :)

### The Problems

**1. Check for Single Even Number**

_Question_:
> Write a function that takes in an integer and returns a Boolean indicating whether or not it is even.

_Solution_:
Just check if `0` is the remainder of the function’s arguments divided by `2`:

{% highlight scala %}
def checkSingleEven(num:Int): Boolean = {
  return (num % 2 == 0)
}
{% endhighlight %}

**2. Check for Even Numbers in a List**

_Question_:
> Write a function that returns True if there is an even number inside of a List otherwise, return False.

_Solution_:
Same logic with the previous problem, but now we iterate through the list’s elements and apply the remainder check for each:

{% highlight scala %}
def checkEvensInList(numList:List[Int]): Boolean = {
  for (num <- numList) {
    if (num % 2 == 0) {
      return true
    }
  }
  return false
}
{% endhighlight %}

**3. Lucky Number Seven**

_Question_:
> Take in a list of integers and calculate the sum of its elements. However, 7 is considered a “lucky number” and they should be counted twice, meaning their value is becoming14 in the calculation. Assume the List is not empty.

_Solution_:
Make a variable to store the total sum, iterate the list’s elements and if 7 is encountered, double the value before addition to the total sum, otherwise just add normally:

{% highlight scala %}
def calcListSum(numList:List[Int]): Int = {
  var sum = 0
  for (num <- numList) {
    if (num == 7) {
      sum += num * 2
    }
    else {
      sum += num
    }
  }
  return sum
}
{% endhighlight %}

**4. Can you Balance?**

_Question_:
> Given a non-empty list of integers, return True if there is a place to split the list so that the sum of the numbers on one side is equal to the sum of the numbers on the other side. For example, given the `List(1,5,3,3)` would return `True` because we can split it in the middle. Another example is `List(7,3,4)` that would also return `True` because `7 = 3 + 4`. Only the boolean value needs to be returned, not the split index point.

_Solution_:
Here, we utilize list’s `splitAt(n)` method: First, make an integer iterator `i` starting from `0`. In every iteration, split the list at index `i` and check if the sum of `leftList` equal to `rightList`, if true then exit the for loop, otherwise continue until `list.length`.

{% highlight scala %}
def checkBalance(numList:List[Int]): Boolean = {
  for (i <- 0 to numList.length) {
    var (left, right) = numList.splitAt(i)
    if (left.sum == right.sum) {
      return true
    }
  }
  return false
}
{% endhighlight %}

**5. Palindrome Check**

_Question_:
> Given a String, return a boolean indicating whether or not it is a palindrome (words spelled the same forwards and backwards, e.g. “ABBA”)

_Solution_:
Here, I am trying not to utilize `str.reverse` built-in method. Instead, i will set two cursors, one in the front and one in the back, and then use a loop to check the equality of the anterior and posterior characters by subsequently move the cursors one step (forward and backward, accordingly) on each iteration and re-do the conditional checks. If the cursors meet in the middle, then the string is deemed as a palindrome.

First, make the string argument into lowercase and subsequently into a list of characters using `str.toLowerCase.toList`. Then set the first index to `0` and last index to length of aforementioned list subtracted by `1`. If the length of list is less than `2`, then return false since palindromes must have at least 3 characters.

I am using two check conditions here: when the length of the string is of an even and odd number. The difference lies in the condition when the cursors meet, in the even case, when the anterior cursor sits on the preceding index of the posterior cursor, then I consider them as “meet in the middle”, whereas in the odd case, it’s when both cursors lie on the same list’s index.

{% highlight scala %}
def checkPalindrome(str:String): Boolean = {
  var strList = str.toLowerCase.toList // handle mixed case
  var firstIndex = 0
  var lastIndex = (strList.length - 1)
  var index = 0
  if (strList.length < 2) {
    println("Palindrome string must be more than 2 characters!")
    return false
  } else if (strList.length % 2 == 0) {
    while (index < strList.length) {
      if ((index + 1) == (lastIndex - index)) {
        return true
      }
      if (strList(index) != strList(lastIndex - index)) {
        return false
      }
      index += 1
    }
    return true
  } else if (strList.length % 2 != 0) {
    var middle_index = (strList.length - 1) / 2
    while (index < strList.length) {
      if (index == middle_index) {
        return true
      }
      if (strList(index) != strList(lastIndex - index)) {
        return false
      }
      index += 1
    }
  }
  return false // catchall
}
{% endhighlight %}

### Conclusion
After working on some exercises, I consider Scala a great language and tool to have in a software engineer's toolbox. It is versatile language which has access to the huge Java ecosystem, while at the same it also has its own set of sophisticated libraries and frameworks. Lastly, for engineers coming from a "higher-level" language like me, Scala allows me to gradually move from simple concepts to more advanced, some of which are not available in most other languages.