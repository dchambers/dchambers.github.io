---
published: false
layout: post
title: "Databases considered harmful"
modified:
categories: articles
excerpt:
tags: [redux, immutable.js, databases]
image:
  feature:
date: 2015-11-07T13:18:09-00:00
---

I've always had a dislike of databases, and I've always had this feeling that the best thing would be to not have a database at all, and to just let the program memory be the database, since modifying program state is an order of magnitude simpler than having to do database queries/updates, and having to shuttle program state back and forth between the client program and the database.

Unfortunately, I could never figure out how to solve the problems with that simpler approach, until I recently found these four separate resources that inspired me:

  * [Turning the database inside-out with Apache Samza](http://www.confluent.io/blog/turning-the-database-inside-out-with-apache-samza/)
  * [The Value of Values with Rich Hickey](https://www.youtube.com/watch?v=-6BsiVyC1kM)
  * [Simplicity Matters by Rich Hickey](https://www.youtube.com/watch?v=rI8tNMsozo0)
  * [Full-Stack Redux Tutorial](http://teropa.info/blog/2015/09/10/full-stack-redux-tutorial.html)

To my surprise, when I subsequently asked the question, "What database would I pair with a full-stack Redux application?", I came to the conclusion: "We don't need databases any more, we can just use the Immutable.js state atom!".

While I highly recommend watching the excellent videos and following the tutorial that led me to this conclusion, the pertinent bits as far as this article are concerned are as follows:

  * We currently build databases around the current state of some data, but the most important information is the original set of events that led to that state (Turning the database inside-out with Apache Samza).
  * Databases should just be a series of cascading materialized views built from the original events (Turning the database inside-out with Apache Samza).
  * Information systems should be built around the processing of facts which are historical records of what happened at a particular time, and are immutable and can't be changed over time (The Value of Values with Rich Hickey).
  * A certain percentage of Big Data is probably businesses telling programmers "that database you have is better than the one you gave me, because the one that you gave me only remembers the last thing I told it." (The Value of Values with Rich Hickey).
  * Simplicity is always preferable to complexity, and relational databases and ORMs are much more complex than just having access to the data, and immutable data is much simpler than mutable data (Simplicity Matters by Rich Hickey).
  * Redux applications have only a single variable containing a reference to the state tree, that is updated in steps by processing a set of actions (Full Stack Redux Tutorial).
  * Full-Stack Redux applications can share large parts of the state-tree between the client and the server, where the server just processes the series of events sent by the various clients, and where the client receives state snapshots from the server as the state changes due to the processing of events. (Redux Full Stack Tutorial).

Now, if you consider the ImmutableJS state-tree to be the materialized view(s) of the historic actions received by an application (given that all state mutation occurs via actions in a Redux application), then our program state is directly comparable to a database, provided that the client events that caused the state tree to have its current value have been persisted.

This approach has a number of interesting properties:

  1. Back-up is just a case of writing the event stream to a replicated disk store.
  1. Replication is just a question of having multiple boxes applying the same event stream, but where only the active box sends state updates to the clients.
  1. When a bug that produced spurious or invalid data in the state tree is fixed, it becomes possible to re-run the event stream with the fix in place, 'healing' the database.
  1. Backwards compatibility of new code can be further verified by replaying the entire set of historic events and confirming that the serialized form of state-tree is equivalent to what it is with the old code.
  1. If a system is hacked, but the event stream is also written to an un-hackable write-only data store, then the database can be more easily healed by removing or amending the hackers events from the event stream, and rebuilding the database using the remaining events.

But also, this is a better approach because there's an order of magnitude of efficiency gains to be had for the parts of the code that are related to server-side state management, and who doesn't want to be the superstar developer that got the job done way quicker than his peers?
