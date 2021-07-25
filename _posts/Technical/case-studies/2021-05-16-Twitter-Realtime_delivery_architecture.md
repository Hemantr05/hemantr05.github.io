---
layout : post
title : "Twitter - Realtime delivery"
date : 2021-05-16
categories : Technical/case-studies
---


### Timeline delivery 

When it comes to tweet delivery, it is really all about timeline.
Now timeline is a chronologically reverse ordered series of tweets.

The following table summarized a few of the timelines used at twitter


|                                   |       Pull                        |         Push                               |
|:----------------------------------|:----------------------------------|:-------------------------------------------|
|Targetted<br>Timeline              |  twitter.com<br>home_timeline API |  User/Site Stream<br>Mobile Push (SMS,etc).|
|Queried<br>Timeline.               |  Search API                       |  Track/Follow Streams.                     |


The targeted timeline is designed for user. When the user logs into twitter.com, the browser "pulls" the home-timeline using the home_timeline API, which is personalized for the user. 
When a user tries to search for something on twitter using keywords, what he/she is doing is accessing the queried timeline using the Search API

Similarlly, push delivery deals capturing user data based on the tokens (users, pages, etc..) the user is interested in using queried timeline. 


When users tweet, they're filling their so-called user-timeline.
Twitter doesn't show a merged-timeline. Now what is a merged-timeline?
Let's take a example, 
Say if a user follows Barack Obama, Elon Musk and Narendra Modi, the tweets of the above personalities will not appear chronologically, with respect to time. As a matter of fact, re-tweets, replied to @randomuser, etc.. play an important role inorder to display content on a user's timeline.

### How does twitter deal this?

A tweet made by a user will first the so-called ***write API*** (an HTTP endpoint). To make sure all of the user's followers views the tweet in realtime, the tweet is first written to a disk and a quick HTTP response will be returned to the client, so everything is asychronous to allow users with variable internet connections stay with the current ongoings as well, after looking for duplicate tweets and formats in the HTTP requests so as to deliver the response in 50 milliseconds.

After the HTTP response, the tweet is fanned out to the hometimelines of all the followers of the user (author of the tweet).

![alt text](https://github.com/Hemantr05/hemantr05.github.io/blob/new_portfolio/assets/img/twitter_entire-arch.jpeg?raw=true)


The timeline cache is based on a bank of Redis instances (in-memory, key-value storage), which unlike memcache (allows binary blobs) deals with the structure of the data.

During ***fanout*** a social graph service, which maintains the information of who follows who. This service finds the followers of the author and then starts the insert process into the Redis instances. 

**Note**: Fanout happens on a bank of machine with a bank of redis instances running, along with multiple TCP connections being open at the same time.

Twitter doesn't cache timeline for inactive users, as redis is an in-memory tool due which it discards old tweets to make room for new ones.


## How are tweets stored?

If we have 5 tweets, the entire tweet would not be stored as it would be hugely redundant. Say the tweet were re-tweeted 'n' times, it would have to stored 'n' times. To avoid that the tweet would be stored in the following fashion.

|          |         |      |
|:--------:|:-------:|:----:|
| Tweet ID | User ID | Bits |
| Tweet ID | User ID | Bits |
| Tweet ID | User ID | Bits |
| Tweet ID | User ID | Bits |
| Tweet ID | User ID | Bits |

As Redis is pretty good at storing data with variable lengths, the attributes (Tweet ID, User ID, Bits) do not necessary have to be of equal length. 


In case of re-tweets, each re-tweet is given it's own ID which is then linked to the original tweet ID.

|          |         |      |          |
|:--------:|:-------:|:----:|:---------|
| Tweet ID | User ID | Bits |          |
| Tweet ID | User ID | Bits |          |
| Tweet ID | User ID | Bits | Tweet ID |
| Tweet ID | User ID | Bits |          |
| Tweet ID | User ID | Bits | Tweet ID |


To pull the tweet to the browser, there exists a ***timeline service***, which will perform a read-operation over the Redi-instances. 


**Note:** It takes about 3 seconds for timeline service to retrieve information for inactive users. It interacts with the social graph service then reads from the redis instances and allow the browser to pull it.

**Timeline caching is expensive on the write and cheap on the read**

## How does the Search API work?

The write API forks onto Ingestor which tokenizes the tweet, geolocation, etc and pushed it into a index in a Indexer - ***Earlybird***, while the tweets are being fanned-out, and writing into redis.
**Blender** an ingeninous tool for seaching indexing, performs a scatter-gather operation on the earlybird instances, along with the user's information, merges and ranks the information and then renders out the resultant tweet, based on user's search

**Search is cheap on the write and more expensive on the read**

## HTTP PUSH / hosebird

* Maintains persistent connections with end clients

* Processes tweet & social graph events

* Event-based "router"

The write API, writes to Hosebird, which collects the tweets and routes them into different queues for public tweets, protected tweets, social events, etc.

To deal the bandwidth of tweet traffic (~1M tweets), **event cascading** is performed, as it is not possible to do so on a single machine.
Event Cascading is fanning out information to multiple machines, which further fan out information to further more machines. 
Twitter's Hosebird cluster consists of 4 layers of cascading. 

**Fact:** Socal graph of a user changes 10 times more often than actual tweets


![alt text2](https://github.com/Hemantr05/hemantr05.github.io/blob/new_portfolio/assets/img/twitter_push-arch.jpeg?raw=true)



**firehose:**

* Edge machine simply outputs the public tweet queue

* Only allow a limited number of firehoses per hosebird box for bandwidth management.


**Track/Follow:**

* Simple query based on tweet content 

* Aka, search lite - as it searches the tweet deck based on user's corpus of interested tokens and then outputs results

* parses publice tweets at the edge, and if term matches a token, or user is of interest, then route.


**User streams:**

* Replicate home timeline experience on user's device

* Upon login, obtain "following" list

* Keep cached following list coherent by seeing social graph updates (if user follows another user(s) from another devices)

* route tweet if from a followed user


## References:

1. [The Engineering Behind Twitter’s New Search Experience](https://blog.twitter.com/engineering/en_us/a/2011/the-engineering-behind-twitter-s-new-search-experience.html)

2. [Real-Time Delivery Architecture at Twitter](https://www.youtube.com/watch?v=J5auCY4ajK8&t=2s)



