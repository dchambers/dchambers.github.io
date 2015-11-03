---
layout: post
title: "Hot-Reloading, Time-Travel & State Serialization"
modified:
categories: blog
excerpt:
tags: [javascript, state-tree, redux, cerebral, baobab]
date: 2015-11-03T22:00:00-00:00
---

These are just three of the amazing properties of State-Tree MVC apps built with frameworks like:

  * [Redux](https://github.com/rackt/redux)
  * [Cerebral](christianalfoni.com/cerebral/)
  * [Baobab](https://github.com/Yomguithereal/baobab)

Actually though, the most practical benefit of these frameworks is the fact that your model, view & controller code becomes purely functional; you push data in, you get data out. Such apps tend to either work or they don't work, are easy to reason about, trivial to unit test, and robust beyond belief.

Here's the quote that blew my mind:

> **Question**: How many variables do you need in a Redux application?

> **Answer**: One. The one inside the store.

Here are some amazing resources to get you up to speed:

  * [Live React: Hot Reloading with Time Travel](https://www.youtube.com/watch?v=xsSnOQynTHs) (the conference talk about the workflow that led to Redux)
  * [Cerebral Introduction Video](https://www.youtube.com/watch?v=O_fk8jBtKSU) (an excellent account of the gradual evolution from vanilla MVC to State-Tree MVC, and the problems that were solved at each step)
  * [Full Stack Redux Tutorial](http://teropa.info/blog/2015/09/10/full-stack-redux-tutorial.html) (an awesome step-by-step tutorial culminating in the creation of a very impressive State-Tree MVC app)

and finally, some food for thought from the Cerebral web-site:

<div style="text-align:center"><img src ="../Cerebral-MCV.png" alt="Cerebral MCV Diagram &mdash; as opposed to MVC" /></div>

Enjoy!
