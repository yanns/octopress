---
layout: post
title: "goto conference 2014"
date: 2014-11-10 10:33:59 +0100
comments: true
categories:
---

My notes from the [goto conference Berlin 2014](http://gotocon.com/berlin-2014):

## Thursday

##### [Opening Keynote: Software Design in the 21st Century](http://gotocon.com/berlin-2014/presentation/Opening%20Keynote:%20Software%20Design%20in%20the%2021st%20Century) - [Martin Fowler](https://twitter.com/martinfowler)

We, developers, take responsibility in the code we write.<br/>
We cannot simply say: "I implemented that because I was told so".<br/>
Avoid [dark patterns](http://darkpatterns.org/).

We are not code monkeys.



##### [Aeron: The Next Generation in Open-Source High-Performance Messaging](http://gotocon.com/berlin-2014/presentation/Aeron:%20The%20Next%20Generation%20in%20Open-Source%20High-Performance%20Messaging) - [Martin Thompson](https://twitter.com/mjpt777)

[slides](http://gotocon.com/dl/goto-berlin-2014/slides/MartinThompson_AeronTheNextGenerationInOpenSourceHighPerformanceMessaging.pdf)
[blog](http://mechanical-sympathy.blogspot.com/)

[Aeron](https://github.com/real-logic/Aeron) is a OSI layer 4 Transport for message oriented streams.
is simply impressive, achieving a very low latency.<br/>
I'd like to see an integration of Aeron in Akka cluster.

[similar talk](https://www.youtube.com/watch?v=tM4YskS94b0)

##### [Writing highly Concurrent Polyglot Applications with Vert.x](http://gotocon.com/berlin-2014/presentation/Writing%20highly%20Concurrent%20Polyglot%20Applications%20with%20Vert.x) - [Tim Fox](https://twitter.com/timfox)

It was a good introduction to Vert.x.<br/>
But I was expecting more than an introduction.

##### [Fast Analytics on Big Data](http://gotocon.com/berlin-2014/presentation/Fast%20Analytics%20on%20Big%20Data) - Petr Maj and Tomas Nykodym
[slides](http://gotocon.com/dl/goto-berlin-2014/slides/PetrMaj_and_TomasNykodym_FastAnalyticsOnBigData.pdf)

Presentation of an ML runtime for bigdata.

[Code](https://github.com/0xdata/h2o) [With Spark API](https://github.com/0xdata/h2o-dev)

TODO: have a look at https://github.com/0xdata/h2o-training

##### [Security Architecture for the SmartHome](http://gotocon.com/berlin-2014/presentation/Security%20Architecture%20for%20the%20SmartHome) - [Jacob Fahrenkrug](https://twitter.com/jacobfahrenkrug)
[slides](http://gotocon.com/dl/goto-berlin-2014/slides/JacobFahrenkrug_SecurityArchitectureForTheSmartHome.pdf)

A good presentation about the future threats of connected objects and the solution [yetu](http://www.yetu.com/) implements.

##### [Graph All The Things!! Graph Database Use Cases That Aren't Social](http://gotocon.com/berlin-2014/presentation/Graph%20All%20The%20Things!!%20Graph%20Database%20Use%20Cases%20That%20Aren't%20Social) - [Emil Eifrem](https://twitter.com/emileifrem)
[slides](http://gotocon.com/dl/goto-berlin-2014/slides/EmilEifrem_GraphAllTheThingsGraphDatabaseUseCasesThatArentSocial.pdf)

very good presentation of [Neo4j](http://neo4j.com/).<br/>
We should use a graph database where it make sense - the search queries are much easier to write / read, and the performance very good.

##### [Party Keynote: Staying Ahead of the Curve](http://gotocon.com/berlin-2014/presentation/Party%20Keynote:%20Staying%20Ahead%20of%20the%20Curve) - [Trisha Gee](https://twitter.com/trisha_gee)
[slides](http://gotocon.com/dl/goto-berlin-2014/slides/TrishaGee_PartyKeynoteStayingAheadOfTheCurve.pdf)

## Friday

##### [Morning Keynote: Excellence Culture & Humane Keeping of Techies](http://gotocon.com/berlin-2014/presentation/Morning%20Keynote:%20Excellence%20Culture%20&%20Humane%20Keeping%20of%20Techies) - [Prof. Dr. Gunter Dueck](https://twitter.com/wilddueck)

a very good keynote, very funny and true at the same time.

##### [Aerospike: Flash-optimized, High-Performance noSQL database for All](http://gotocon.com/berlin-2014/presentation/Aerospike:%20Flash-optimized,%20High-Performance%20noSQL%20database%20for%20All) - [Khosrow Afroozeh](https://twitter.com/khaf)
[slides](http://gotocon.com/dl/goto-berlin-2014/slides/KhosrowAfroozeh_AerospikeFlashOptimizedHighPerformanceNoSQLDatabaseForAll.pdf)

A new database on my radar - impressive.


##### [Docker - A Lot Changed in a Year](http://gotocon.com/berlin-2014/presentation/Docker%20-%20A%20Lot%20Changed%20in%20a%20Year) - [Chris Swan](http://twitter.com/cpswan)
[slides](http://gotocon.com/dl/goto-berlin-2014/slides/ChrisSwan_DockerALotChangedInAYear.pdf)

Docker is maturating.<br/>
Just do not forget that "containers do not contain" -> no complete security between container and host, and between containers.


##### [The Joys and Perils of Interactive Development](http://gotocon.com/berlin-2014/presentation/The%20Joys%20and%20Perils%20of%20Interactive%20Development) - [Stuart Sierra](https://twitter.com/stuartsierra)
[slides](http://gotocon.com/dl/goto-berlin-2014/slides/protected/StuartSierra_TheJoysAndPerilsOfInteractiveDevelopment.pdf)

##### [New Concurrency Utilities in Java 8](http://gotocon.com/berlin-2014/presentation/New%20Concurrency%20Utilities%20in%20Java%208) - [Angelika Langer](https://twitter.com/AngelikaLanger)
[slides](http://gotocon.com/dl/goto-berlin-2014/slides/protected/AngelikaLanger_NewConcurrencyUtilitiesInJava8.pdf)

- Future API is becoming usable, it is now possible to chain futures and callbacks.
- new StampedLock is optimized for reading. (personal note: but lock-free algorithms should be preferred, like in [Lock-Based vs Lock-Free Concurrent Algorithms](http://mechanical-sympathy.blogspot.com/))

##### [Adaptive Planning - Beyond User Stories](http://gotocon.com/berlin-2014/presentation/Adaptive%20Planning%20-%20Beyond%20User%20Stories) - [Gojko Adzic](https://twitter.com/gojkoadzic)
[blog](http://gojko.net/)
[slides](http://gotocon.com/dl/goto-berlin-2014/slides/GojkoAdzic_AdaptivePlanningBeyondUserStories.pdf)

Gojko Adzic really thinks agile. Good presentation of how to make better user stories.<br/>
Do not describe a desired behavior but a behavior change.