---
layout: post
title: "Databases are dead, long live the immutable state atom!"
modified:
categories: articles
excerpt:
tags: [redux, immutable.js, databases, nodb]
image:
  feature:
date: 2015-11-09T13:23:36-00:00
---

As an app developer, I've always had a dislike of databases, and I've always had this feeling that the best thing would be to not have a database at all, and to just let the program memory be the database, since modifying program state is an order of magnitude simpler than having to do database queries/updates, and having to shuttle program state back and forth between the client program and the database.

## Some Great Resources

Unfortunately, I could never figure out how to solve the obvious problems with that simpler approach, until I recently found these four separate resources that illuminated the path for me:

  * [Turning the database inside-out with Apache Samza](http://www.confluent.io/blog/turning-the-database-inside-out-with-apache-samza/)
  * [The Value of Values with Rich Hickey](https://www.youtube.com/watch?v=-6BsiVyC1kM)
  * [Simplicity Matters by Rich Hickey](https://www.youtube.com/watch?v=rI8tNMsozo0)
  * [Full-Stack Redux Tutorial](http://teropa.info/blog/2015/09/10/full-stack-redux-tutorial.html)

To my surprise, when I subsequently asked the question, "What database would I pair with a full-stack Redux application?", I came to the conclusion: "We don't need databases any more, we can just use the Immutable.js state atom!".

## The Pertinent Bits

While I highly recommend watching the excellent videos and following the tutorial that led me to this conclusion, the relevant bits as far as this article are concerned are as follows...

### Turning the database inside-out with Apache Samza

Points of interest:

  * We currently build databases around the current state of some data, but the most important information is the original set of events that led to that state.
  * Databases should just be a series of cascading materialized views built from the original events.

### The Value of Values with Rich Hickey

Points of interest:

  * Information systems should be built around the processing of facts which are historical records of what happened at a particular time, and are immutable and can't be changed over time.
  * A certain percentage of Big Data is probably businesses telling programmers "that database you have is better than the one you gave me, because the one that you gave me only remembers the last thing I told it.".

### Simplicity Matters by Rich Hickey

Points of interest:

  * Simplicity is always preferable to complexity, and relational databases and ORMs are much more complex than just having access to the data, and immutable data is much simpler than mutable data.

### Full Stack Redux Tutorial

Points of interest:

  * Redux applications have only a single variable containing a reference to an immutable state atom, and this immutable state atom is updated in steps by processing a set of actions.
  * Client/Server Redux applications can share large parts of the state-tree between the client and the server, where the server processes the events sent by the various clients, and where the client receives state snapshots from the server as the state changes due to the processing of those events.

## Putting it all Together

Now, if you consider the server-side immutable state atom to be a materialized view of the historic events received by a server, you can see that we've already got something very close to a Samza style database, but without the event persistence.

Assuming we can do event persistence properly, this approach has a number of interesting properties:

  1. Back-up is just a case of writing the event stream to a replicated disk store.
  1. Replication of the materialized view can be done by having multiple boxes apply the same set of agreed events.
  1. Cascaded materialized views can be achieved using the same state synchronization technique used by the client.
  1. When a bug that produced spurious or invalid data in the state tree is fixed, it becomes possible to re-run the event stream with the fix in place, 'healing' the database.
  1. Backwards compatibility of new code can be further verified by replaying the entire set of historic events and confirming that the serialized form of the state-tree is equivalent with both the new code and the old code.
  1. If a system is hacked, but the event stream is written to an un-hackable _append-only_ data store, then the database can be more easily healed by removing or amending the hackers events from the event stream, and rebuilding the materialized views using this modified set of events.

But also, this is a better approach because there's an order of magnitude of productivity gains to be gained due to the fact you avoid all of the state synchronization complexity by killing of the database!

## The Architecture

This is how I think a Redux nodb application is structured:

<script src="https://www.gliffy.com/diagramEmbed.js" type="text/javascript"></script><script type="text/javascript">gliffy_did="9392265";embedGliffy();</script>

Let's take a look at what these boxes all do...

### Chronicler

The Chronicler forms the accepted history of all client and server events. It only acknowledges receipt of an event after it's been persisted to disk, and it's able to ignore re-sends for events that have already been accepted. Taken together, this allows clients to know if user actions have been received, and to avoid penalization for re-sending if they incorrectly believe an action hasn't been received.

Only one chronicler is active at any time, where an active standby will normally be ready to take ownership of the chronicler's IP address should the heartbeat from the active chronicler stop.

The chronicler writes the records to a NAS, which is the closest thing to a database in this architecture. By setting the sticky bit on the directory the chronicler writes its records to, it becomes impossible to erase history even if the chronicler is compromised.

### Data View

The Data Views maintains a state-tree based on the event stream from the active chronicler. They are composed of a set of reducer functions and nothing else. In addition to the reducers for the domain actions, data-views must also support two non-domain specific actions:

  1. `SET_STATE` to replace the state-tree with some data provided by the chronicler.
  2. `SNAPSHOT` to serialize the current state-tree to the SAN.

Clients will normally seek to establish a connection to the data-view before connecting to the chronicler, on the basis that clients can't sensibly seek to affect the state-tree until they first know its current state. Here, round robin DNS is used to evenly share the load between the hosts providing the data-views.

For isomorphic apps, the data-view will also effectively be the app-server, as it will need to provide rendered page views for any app URLs, in addition to the streamed state updates.

### Envoy

Envoys are used to communicate to the outside world. Since this involves creating asynchronous actions, some form of Redux middle-ware will normally be used for this purpose. However, the built in Redux middle-ware (e.g. [promises](https://github.com/acdlite/redux-promise), [observables](https://github.com/acdlite/redux-rx), or [thunks](https://github.com/gaearon/redux-thunk)) can't be used because the resultant synchronous actions must be channeled back though the chronicler so they can be persisted.

Envoys are purposely not communicated to when the chronicler is time traveling (e.g. if restoring from a crash by pushing a snapshot and then the actions that came after that snapshot) since this would otherwise cause historic actions to be repeated when they shouldn't be.

### Cascaded Data View

A cascaded data-view provides some derived view based on the contents of one or more up-stream views. They use the same streaming state updates that clients use to access a view. Like normal data views, they are composed from a set of reducer functions and nothing else.

Cascaded data views are designed to be run on separate hosts from the view(s) they are derived from; instead, [reselect](https://github.com/rackt/reselect) can be used if you prefer to have the derived data directly within the primary data-view. So, they are mostly about scalability, but they also have the benefit that they can also be created by third parties since they use the same streaming connection available to clients.

## Scalability

The chronicler is the only choke point in the system, but since they only need to write events to a disk, and since client authentication can be done when the connection is formed, it should be possible to handle very large numbers of clients when using a single-threaded non-blocking server like Node.js. A consequence of this design is that client events should only describe the client's intention, and not it's affect on the world, since the chronicler does not attempt to verify whether events can be processed or not. For example, the event "George swings his sword at the dragon" is not the same as "George slays the dragon" &mdash; unfortunately for the hero, the dragon may have already moved on in the meantime!

One of the nice things about this architecture is the data scalability. Traditional databases always end up becoming coupled to the applications they are designed to support, and when new applications are needed new databases are created whose state must be kept aligned with the original databases, which is in fact an endless stream of bug fixing for a team of DBAs!

On the other hand, events are less coupled to an application. When a new application comes along we may find that certain events start to include additional information that the earlier applications didn't need, and include entirely new events too. But this is fine since the materialized views for the original events can continue to ignore these new events, and any new fields on pre-existing events, yet there is always a single unified event-stream for the entire business, and all of the apps and legacy apps that are created along the way.

## Missing Bits

There are a number of missing bits that, at present, apps would need to implement themselves:

  * The Chronicler is an entirely re-usable component that could be shared by all apps.
  * The full-stack Redux example currently sends the entire state to the client instead of a diff, which is obviously inefficient, so diff/patch primitives would be helpful here.
  * The streaming connection for the data-view probably contains some re-usable bits in addition to the diffing/patching.
  * A reducer for the `SET_STATE` and `SNAPSHOT` actions could be produced.


## Conclusion

In addition to making it possible to do pure functional programming within GUI apps, the same technologies also make it possible to have something far superior to the traditional database. Redux + Immutable.Js + React really are the gifts that keep on giving!
