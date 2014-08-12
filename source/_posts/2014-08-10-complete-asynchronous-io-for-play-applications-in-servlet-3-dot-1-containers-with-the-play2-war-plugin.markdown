---
layout: post
title: "asynchronous IO for Play! applications in servlet 3.1 containers with the play2-war plugin"
date: 2014-08-10 18:55:36 +0200
comments: true
categories:
- Play Framework
- Play2 war plugin
---



A [Play application](http://playframework.com/) does not need any container and [runs directly in production](http://playframework.com/documentation/2.3.x/Production).

However some organizations prefer to run play applications within a servlet container and can use for this the [WAR Plugin](https://github.com/play2war/play2-war-plugin).

I am very pleased to share that this plugin is now compatible with servlet 3.1 containers. It can now use the new asynchronous IO possibilities.

Let me explain why and when it is important.

### Play applications are asynchronous

The Play Framework is build to be totally asynchronous and reactive. It uses no blocking IO. A Play application scales very well, using very few threads.

This reactive consuming or construction of IO stream is designed in play with [Iteratees](http://www.playframework.com/documentation/2.3.x/Iteratees). It will progressively be completed with [Akka Streams](http://typesafe.com/blog/typesafe-announces-akka-streams), the implementation in Akka of the reactive stream project (http://www.reactive-streams.org/)

### Servlet containers use blocking IO

On the other hand, servlet containers traditionally use a [thread per request](http://www.slideshare.net/brikis98/the-play-framework-at-linkedin/62). When doing IO (database access, external web request), the thread waits for the IO completion (blocking IO). That's why servlet containers needs a lot of working threads to handle requests in parallel ([default 200 threads max for tomcat](http://tomcat.apache.org/tomcat-7.0-doc/config/executor.html))

#### Asynchronous 3.0 servlet

Servlet 3.0 containers introduced the possibility to “suspend” a request.
For example, if an application makes an HTTP request to another web service using the [WS client](http://www.playframework.com/documentation/2.3.x/ScalaWS), the play2 war plugin is able to suspend the servlet request until the external web service answers. It means that with the same number of servlet threads, a servlet 3.0 container can support more requests in parallel than a servlet 2.x container. It does not need that a thread waits for an HTTP request, this thread can be used for other requests.

#### Limitations of asynchronous 3.0 servlet

When uploading or downloading a big file, the servlet container is able to stream the data. But between each chunks, the servlet thread is blocking, waiting for the next one.
It is not possible to consume one chunk, and later when we are ready to consume another one, tell the container: “now I am ready, you can send me the next chunk as soon as you receive one from the browser”.

If a browser needs an hour to upload a file with a slow Internet connection, the container needs a thread during an hour, even if the application does not do anything, just waiting for the upload completion.

### Play applications deployed as war are limited by the servlet container

A Play application deployed in a war container are also limited by the technical possibilites of the servlet API. With a servlet 2.x or 3.0, a Play application does not scale as well as when run natively.

### New asynchronous IO in servlet 3.1

The [servlet 3.1 specification](https://jcp.org/en/jsr/detail?id=340) added asynchronous IO. Based on [events](http://docs.oracle.com/javase/7/docs/api/java/util/EventListener.html), it is now possible to completly handle an [upload](https://javaee-spec.java.net/nonav/javadocs/javax/servlet/ReadListener.html) and a [download](https://javaee-spec.java.net/nonav/javadocs/javax/servlet/WriteListener.html) asynchronously.

### Asynchronous IO in WAR Plugin for Play! applications

Since the [version 1.2](https://github.com/play2war/play2-war-plugin/releases/tag/1.2), the play2 war plugin is using this API to provide a complete reactive upload and download.

To use this, simply [configure the servlet container version](https://github.com/play2war/play2-war-plugin/wiki/Configuration#servlet-31-container-configuration) and deploy to a servlet 3.1 server.

#### When to use this feature?

This feature should be used especially if the application is using big download/upload. The servlet 3.1 will help to scale much better.

During my tests, I could upload and download files from several GB in parallel. The container and the JVM garbage collection could support that without any problems. The memory usage was very low.

Also please test this new version and report issues!

#### How to build the application to scale better?

To scale as much as possible, the application should not block. It should always use asynchronous IO API like in the WS client.

But in the Java World, a lot a librairies are still designed for a one-thread-per-request model and do not provide asynchronous API. It is for example the case for a JDBC driver. In that case, a separate dispatcher should be configured to handle especially blocking IO. More Information for this can be found in the [Play Framework documentation](http://www.playframework.com/documentation/2.3.x/ThreadPools).


### Implementation history

The implementation of asynchronous IO in the WAR plugin lasted a few months.
The [first pull request](https://github.com/play2war/play2-war-plugin/pull/204) introduced the asynchronous download, and [the second one](https://github.com/play2war/play2-war-plugin/pull/235) the asynchronous upload.
I'd like to thank [James Roper](https://twitter.com/jroper) and [Viktor Klang](https://twitter.com/viktorklang) for their reviews.

This feature was quite challenging to implement:

- I had to find a good way to implement the glue between two very different APIs. The servlet 3.1 API is quite imperative and use [EventListener](http://docs.oracle.com/javase/7/docs/api/java/util/EventListener.html) and methods with side effects. The Iteratee API is functional and I needed some time to feel at ease with it.

- The servlet 3.1 specification was recently finalized as I began. The first implementations in some containers contained bugs. Reporting the problems, explaining, convincing other developers took a lot of time and energy.

- The servlet 3.1 specification is not always explicit. The implementations in the different servlet containers are also different. Finding an implementation that covers all these subtle differences was challenging. The [testing infrastructure of the play2-war plugin](https://play-war.ci.cloudbees.com/job/Play_2_War_Run_integration_tests_-_Play_22x/) provides a lot of integration tests with different containers and helped a lot for this.

My firm [Leanovate](http://www.leanovate.de/) gave me some time to work on that. Having worked two days full time on it helped me a lot. Thanks Leanovate for this!

All in all it was a great experience to contribute to the WAR Plugin, and I hope my work will be useful for others.
