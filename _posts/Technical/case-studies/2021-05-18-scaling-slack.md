---
layout : post
title : "Scaling Slack"
date : 2021-05-18
categories : Technical/case-studies
---

# How slack works

Slack is mainly known for its **Persistent Group Messaging** service.


## Trivia:

* 4M Daily active user (DAU) and 5.8M WAU

* Slack has about 2.5M web socket connections open simultaneously at their peak.
 
* Half of DAU outside US

* Slack uses a conservative tech-stack: Technologies/tools which are >10years olds

* Developers at Slack choose something they've already operated on, over something new and tailor-made, shallow, transparent stack of abstraction



![alt text](https://github.com/Hemantr05/hemantr05.github.io/blob/new_portfolio/assets/img/slack_arch.jpeg)

**Slack's webapp codebase:**

* PHP monolith of app logic (<1M Lines of Code)

* Scaled-out LAMP stack app (Memcache wrapped around sharded MySQL)

* Recently migrated to HHVM (HipHop Virtual Machine, Facebook's JIT (just-in-time) for PHP)


**Login and Receive Messages: the "mains"**

![the mains](https://github.com/Hemantr05/hemantr05.github.io/blob/new_portfolio/assets/img/slack_the_mains.jpeg)


![the shards](https://github.com/Hemantr05/hemantr05.github.io/blob/new_portfolio/assets/img/slack_the_shards.jpeg)

MySQL Shards:

* Source of truth for most customer data (Teams, users, channels, messages, comments, emoji, ...)

* Replication across two Data Centers (Available for 1-DC failure)

* Sharded by teams (For performace, fault isolation, and scalability)

By why MySQL?

* Many, many thousands of server-years of operating

* The relational model is a good discipline

* **Not because of ACID, though**

How is MySQL used then?

At slack, MySQL is used for ***master-master replication***

![MMR text](https://github.com/Hemantr05/hemantr05.github.io/blob/new_portfolio/assets/img/slack_MMR.jpeg)

This helps in retreiving data from one shard in case of failure of the other, as write operations are performed on both shards simultaneously.


Now what if the same row is written or same value is written ? How does Slack with MMR complications?

* Choosing *A* in CAP theorem. (Availability)

* Conflics are possible:
    * Most resolved automatically (with application knowledge)

* **INSERT ON DUPLICATE KEY UPDATE ...**

* Partitioning by teams saves slack from most of the complications
    * Team writes cannot overlap
    * Even teams use "left" head, odd teams use "right" head


The rtm-API (RealTime Messaging API), does all of the above and shares to the user the following but not limited to:

1. Identity of every channel in a team,

2. ID of every user in the team,

3. The membership of all channels,

4. Where has the cursor moved since you were last active on the channel, etc..

But the two important pieces of information are: 

```
{
    "ok": True,
    "url": "wss:\/\/ms9.slack-msgs.com/\websocket\/7I5yBpcvk",
    ...
}
```

Using the above frame of information of each session, slack manages realtime update with cache memory of the websocket connection

Rtm.start payload

* Rtm.start returns an image of the whole team

* Architecture of clients
    * Eventually consistent snapshot of whole team
    * Updates trickle in through the web socket

* Guarantees responsive clients... once connection is established


**Message Delivery**

![msg delivery](https://github.com/Hemantr05/hemantr05.github.io/blob/new_portfolio/assets/img/slack_msg_delivery.jpeg)

* There exists a race between rtm.start and connection to MS (Message Server)
    * Event log machanism (the hash in the above websocket url, points to the event log)

* Glitchesm delays net partitions while persisting
    * In-memory queue of pending sends
    * Queue depth sensitive barometer of system health

* Most message are presence 


Deferring Work:

![def work](https://github.com/Hemantr05/hemantr05.github.io/blob/new_portfolio/assets/img/slack_def_work.jpeg)

* Slack uses Redis as a Job Queue

* Link unfurling is the mechanism to curl into reference links.

    * For example, user types https://twitter.com/@someuser, link unfurling curls into "someuser"'s twitter handle and render a small box of his/her profile on slack chatbar

 
## How slacks works today

* Team-partitioning
    * Easy scaling to lots of teams
    * Isolates failures and performance problems
    * Makes customer complaints easy to field
    * Natural git for a paid product

* Per-team Message Server
    * Low-latency broadcasts


## Hard scenarios slack deals with

* Mains failures
    * If one master fails, patner takes over
    * What if both fail?
        * Many users can proceed via memcache
        * For the rest Slack is down
        * Quite possible if failure was load-induced

* Rtm.start on large teams
    * Returns image of **entire** team
    * Channel membership is O(n^2) for n users

* Mass reconnects
    * A large team loses, then regains, office Internet connectivtity
    * n users perform O(n^2) rtm.start operations
    * Can 'melt' the team shard

## Solutions for the above problem?

* Scale-out mains
    * Replace mains spof (single point of failure)

* Rtm.start for large teams
    * Incremental work
    * Core problem: channel membership is O(n^2)
        * Change APIs so clients can load channel members lazily (much harder than it sounds!)

* Mass reconnects
    * Introducing flannel (an application-level edge cache)


