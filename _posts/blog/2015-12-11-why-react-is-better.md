---
layout: post
title: "Why React is Better"
modified:
categories: blog
excerpt:
tags: [javascript, mvc, react, redux]
date: 2015-12-11T21:00:00-00:00
---

I'm writing this very quick blog because I've had a few conversations recently where I've had to explain to people why React is not only better, but more or less the final story as far as JavaScript view libraries go; as final as anything really gets on the web anyway. And, in 2015, when all the thought leaders have adopted React, it's surprising that you still often encounter scepticism when you say this, and we still see articles like this:

  * [React Is A Terrible Idea](https://www.pandastrike.com/posts/20150311-react-bad-idea)
  * [Why I Like Angular More Than I Like React.JS](http://fizzylogic.nl/2015/06/10/why-i-like-angular-more-than-i-like-react-js/)
  * [Pros and Cons of Facebook's React vs. Web Components (Polymer)](http://programmers.stackexchange.com/questions/225400/pros-and-cons-of-facebooks-react-vs-web-components-polymer)

So, although I don't really have the time to make this a full article where I justify all my points, I'd just like to get these points down, and they can be like a conversation starter or research starters for anyone that does happen to read this.

Okay, here goes:

  * It allows the entire view to be rendered declaratively, avoiding all the observers, and the complex stateful update code you usually have to deal with.
  * It doesn't allow the view to have any awareness of the model since all data is pushed in as props, allowing views to be re-used with different sources of data, and tested without a real back-end, with no need for mocking or IoC mechanisms.
  * It allows granular re-use since views are composed of components, and you are free to re-use only the components you're interested in.
  * It co-locates the tags and the code, removing a potential source of bugs and further reducing cognitive load.
  * It allows uni-controller models (like Flux, Cerebral and Redux) that allow all actions to be dispatched to a single point, creating a single direction of flow through the code.
  * It allows completely stateless, pure functional views, where the state can be held externally, so that views become trivial to reason about and test.
  * When paired with something like Redux, which allows all state to be kept in one place as a single immutable state atom (aka a state-tree), and which allows the controller to be defined as set of pure functions that create an updated state given an earlier state and action to perform, the entire code-base becomes purely functional.

A magical thing happened when I created my first pure functional React app. Although the program wasn't obviously quicker to create than with any other comparable technology, once it worked, it really just worked, and I didn't subsequently find a single edge case bug. That's never happened to me before except for quite small bits of code I've written. The significance of that experience wasn't lost on me!

Then, yesterday, when somebody at work linked David Lubar's humorous [Program Development Cycle](http://www.math.psu.edu/tseng/progcycle.html):

> 1. Programmer produces code he believes is bug-free.
> 1. Product is tested. 20 bugs are found.
> 1. Programmer fixes 10 of the bugs and explains to the testing department that the other 10 aren't really bugs.
> 1. Testing department finds that five of the fixes didn't work and discovers 15 new bugs.
> 1. See 3.
> 1. See 4.
> 1. See 5.
> 1. See 6.
> 1. See 7.
> 1. See 8.
> 1. Due to marketing pressure and an extremely pre-mature product announcement based on over-optimistic programming schedule, the product is released.
> 1. Users find 137 new bugs.
> 1. Original programmer, having cashed his royalty check, is nowhere to be found.
> 1. Newly-assembled programming team fixes almost all of the 137 bugs, but introduce 456 new ones.
> 1. Original programmer sends underpaid testing department a postcard from Fiji. Entire testing department quits.
> 1. Company is bought in a hostile takeover by competitor using profits from their latest release, which had 783 bugs.
> 1. New CEO is brought in by board of directors. He hires programmer to redo program from scratch.
> 1. Programmer produces code he believes is bug-free.
> 1. See step 2

it immediately occurred to me that this was the perfect exemplum for why React is better. Because kids, pure functional programming significantly reduces the number of edge-case bugs you'll find in production code, and that's just the most compelling of many reasons why React is better.
