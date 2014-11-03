---
layout: post
title: "On the discovery of IterizerJs"
modified:
categories: blog
excerpt:
tags: [javascript, es6, iterators, generators, range]
date: 2014-10-06T09:00:00-00:00
---

A couple of months ago a friend of mine told me about how he now, where possible, writes JavaScript using the [ES5 array methods](), avoiding loops whenever he can. He claimed that programming like this leads to more concise code with fewer "dangling variables" \[sic\] &mdash; a broken metaphor derived from "dangling pointers" perhaps, but one which struck a chord with me anyway. Intrigued, I decided to do what I did last time I wanted to fully grok a new programming paradigm; I took the _Project Euler Challenge_.

For those of you that haven't already heard of it, [Project Euler](https://projecteuler.net/) is a series of logical programming challenges, growing in complexity as you progress from one problem to another. It's a great way to become better at solving logical conundrums, but it's also a nice way to gain mastery in new programming paradigms, like [functional programming](http://en.wikipedia.org/wiki/Functional_programming).

The very first problem in Project Euler is:

> Find the sum of all the multiples of 3 or 5 below 1000.

Straight away it becomes obvious that none of the ES5 array methods are going to help with problems like this that are all about numbers. What we need is the JavaScript equivalent of Python's `range()` function. But it itself is dependent on Python's support for iterators and generators.

## ES6 to the Rescue

As it happens, ES6 adds support for both iterators and generators, and it's already supported in the latest versions of Chrome, Firefox & Node.js. Now we can have a Python style `range()` function!

Because Python's `range()` function is so useful, there were actually a number of implementations on the web already, so I grabbed one and continued on my way. With it, the solution to the first problem became:

``` javascript
Array.from(range(999)).filter(function(n) {
	return (((n % 3) == 0) || ((n % 5) == 0));
}).join(', ');
```

This isn't too pretty, and I don't feel like I'm writing more concise code. I feel like I'm writing ugly, hard to read code. The problem is ES6's `Array.from()` method, because it's a static method that I have to provide the context to, rather than being immediately available on all _iterable_ objects.

To make myself feel cleaner again, I patch `Array.from` directly onto `Object.prototype.toArray` as a non-enumerable property, and enjoy the improved view:

``` javascript
range(999).toArray().filter(function(n) {
	return (((n % 3) == 0) || ((n % 5) == 0));
}).join(', ');
```


## The Array.from() bridging anti-pattern

As I work through the various problems, I often have to use `Array.from()` (via `Object.toArray()` in my case) to be able to use the ES5 array methods with _iterables_. Apart from being wasteful in terms of memory usage, it becomes apparent that doing so reduces the re-use potential of functions that could otherwise themselves have returned _iterables_.

The first example of this is with prime numbers, which feature in both questions 3 and 7. To solve these problems in ES5 we might end up creating `primes(numPrimes)`, `primes(maxPrime)` & `nthPrime(n)` methods, whereas with ES6 generators we can theoretically create a single `primes()` method that returns primes indefinitely, and achieve the variations using:

  * `primes().limit(numPrimes)`
  * `primes().limit(lessThanOrEqualTo(maxPrime))`
  * `primes().nthPrime(n)`

increasing re-use at the functional level.

When we later encounter problem 10 from Project Euler ('Find the sum of all the primes below two million'), our intuitions are confirmed when we can solve the problem with this single line of code:

``` javascript
primes().limit(lessThan(2000000)).sum();
```

The downside here being, when it becomes this simple to solve the problems, it becomes no challenge at all!

## Conclusion

ES6 generators and iterators make one of Python's best features available for use in JavaScript. By adding _iterable_ versions of the ES5 array methods to the mix, along with a `range()` and a `limit()` function, [IterizerJs](https://github.com/dchambers/iterizerjs) makes it easy to write code that is both expressive and concise.
