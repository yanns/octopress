---
layout: post
title: "trampoline execution context with scala Futures"
date: 2016-01-19 08:31:33 +0100
comments: true
categories: 
---

Disclaimer: I am continually learning, and this post reflects my current understanding of the topic. I may be wrong. Do not believe directly what I write. Test what you need. If you want to provide me some precisions, some corrections, please contact me on [twitter](https://twitter.com/simon_yann), and I'll fix this post.

[Rúnar](https://twitter.com/runarorama) showed in a [blog post how Scalaz Tasks can perform better than the Scala Futures](http://blog.higher-order.com/blog/2015/06/18/easy-performance-wins-with-scalaz/).

He explains the details very well. If you have not read that post, I recommand it highly.

The main point if that Scala `Future` adds a context switching for each `map` or `flatMap`.
With Scalaz `Task`, we have to describe which tasks need a new thread, the other ones are called on the same thread as the previous computation, avoiding these context switchings.

With Scala Futures, if you want to multiply the result of a `Future[Int]` by 2, you need an `ExecutionContext:
```scala
import scala.concurrent.ExecutionContext.global
val futureCount: Future[Long] = futureCountOfUsers()
val result = futureCount.map(i => i * 2)(global)
```
or with an implicit resolution:
```scala
import scala.concurrent.ExecutionContext.Implicits.global
val futureCount: Future[Long] = futureCountOfUsers()
val result = futureCount.map(i => i * 2)

```

To compute the `i => i * 2`, the ExecutionContext may use a different thread than the one having the result of the `futureCountOfUsers`. We observe a context switching between the future and the callback in the `map`. And the thread executing `i => i * 2` can run on a different CPU/core than the one having the result of `futureCount`, meaning that the CPU cache is missed.

This overhead is not problematic for simple computations. But if we do 100 or 1000 of them, then it can have a significant impact on performances.

And my opinion, Scala Futures have other problems.

For example, with the following code:
```scala
for {
  value1 <- functionThatReturnsFutureValue1
  value2 <- functionThatReturnsFutureValue2
} yield (value1, value2)
```
`functionThatReturnsFutureValue1` and `functionThatReturnsFutureValue2` runs sequentially, even if there is no dependency between the two.

On the other hand:
```scala
val futureValue1 = functionThatReturnsFutureValue1
val futureValue2 = functionThatReturnsFutureValue2

for {
  value1 <- futureValue1
  value2 <- futureValue2
} yield (value1, value2)
```
computes `functionThatReturnsFutureValue1` and `functionThatReturnsFutureValue2` in parallel.

It means that Scala Futures do not respect the principe of ["referential transparency"](https://en.wikipedia.org/wiki/Referential_transparency).
It's not only a theoretical problem, but bits a lot of new users of Scala Futures.


And what I do not like about Scala Future is that we always need an ExecutionContext, even for small non-blocking computations.

For example, instead of writing:
```scala
def multiplyBy2(f: Future[Long]): Future[Long] =
  f.map(_ * 2)
```

We need either to import a ExecutionContext that is always used, or leave the liberty to the caller of the function and write:
```scala
def multiplyBy2(f: Future[Long])(implicit ec: ExecutionContext): Future[Long] =
  f.map(_ * 2)
```

My first impression with Scalaz Tasks is that they have a better design than the Scala Futures.
But I have not used Scalaz Tasks extensively and cannot say if they have other problems.

All in all, Scala Futures are here to stay. They are part of the standard API and are used everywhere.

I'm still wondering why the Scala Futures were designed that way.
I can only guess, but I think this avoids some common pitfalls:

- "easy" for new-comers: simply import the default execution context and that’s all
- safe by default: If a callback takes a long time (blocking IO and expensive computation), this callback will not block other ones. The execution context will be able to use a different thread for other computations.
- and a design like Scala Tasks works well if all parts of the system are non-blocking and using one thread pool. The reality is more complex. Each driver/http client can use its own thread pool. For example, an asynchronous http client may have its own thread pool because some parts of the networking API in Java is blocking like the standard ns lookup `InetAddress.getByName()`. Running a computation directly on the thread without forking it will run the computation of the thread pool of the http client. And that can lead to an exhaustion of the thread pool of the http client, and the http client cannot function anymore.


### Introducing the trampoline EC

This performance problem with the standard EC is not new. The play framework team had this problem, especially with Iteratees that compute everything with a Future and uses callbacks extensively on stream of data.
To solve this problem, James Roper introduced the [trampoline Execution Context](https://github.com/playframework/playframework/blob/master/framework/src/iteratees/src/main/scala/play/api/libs/iteratee/Execution.scala#L31-L128).
This trampoline EC is a great piece of software:

- it makes sure the callbacks are run on the same thread than the future to avoid context switchings.
- it does not overflow with recursive callbacks.


To show the benefit of the trampoline execution context, let's call this function that does not make any sense, but simply calls `Future.map` n times:
```scala
def range(n: Int)(implicit ec: ExecutionContext): Future[Int] =
  (0 to n).foldLeft[Future[Int]](Future.successful(0)) { case (f, _) ⇒ f.map(_ + 1) }
```

With n = 5 000 000:

- Scala Futures with standard EC: 0.037 ops/s
- Scala Futures with trampoline EC: 1.397 ops/s


### Should we use the trampoline EC?

If you are quite confident with EC, then you can use this trampoline EC if:

- you are sure that the callback is running very fast. For example:
```scala
def multiplyBy2(f: Future[Long]): Future[Long] =
  f.map(_ * 2)(trampolineEC)
```
- you never call some blocking IO. This point can be tricky: some scala libs can use some java libs that use InputStream or OutputStream that can block.

If you are unsure, use the standard EC.


If you want to try this yourself, the [code is on github](https://github.com/yanns/trampoline-EC).

