---
layout: post
title: "SLF4J Mapped Diagnostic Context (MDC) with play framework"
date: 2014-05-04 11:45:52 +0200
comments: true
categories:
- Play Framework
- MDC
- SLF4J
---

I'd like the share with this post one solution I found to use a Mapped Diagnostic Context (MDC) in an asynchronous environment like the play framework.

## tl;dr

This solution uses a custom `ExecutionContext` that propagates the MDC from the caller's thread to the callee's one.
A custom `ActionBuilder` is necessary as well to completely use this custom `ExectionContext`.

## The Mapped Diagnostic Context (MDC)

The play framework uses for logging [Logback](http://logback.qos.ch/) behind [SLF4J ("Simple Logging Facade for Java")](http://www.slf4j.org/).<br/>
This library provides a convenient feature: the [Mapped Diagnostic Context (MDC)](http://logback.qos.ch/manual/mdc.html).
This context can be used to store values that can be displayed in every Logging statement.<br/>
For example, if we want to display the current user ID:
```scala
    import org.slf4j.MDC

    val id = currentUser.id
    MDC.put("X-UserId", currentUser.id)

    try {
      // the block of code that uses the Logger
      // for example:
      play.api.Logger.info("test")
    } finally {
      // clean up the MDC
      MDC.remove("X-UserId")
    }
```
(This code could be in a [filter](http://www.playframework.com/documentation/2.2.x/ScalaHttpFilters), run for each request)

Logback must be configured to display the `X-UserId` value:
```xml
    <appender name="stdout" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} %coloredLevel %logger{35} %mdc{X-UserId:--} - %msg%n%rootException</pattern>
        </encoder>
    </appender>
```
In the log file, the MDC value for `X-UserId` is now printed out.
```
10:50:54.773 [info] application jean.leloup - test
```

## Limitation of the default implementation of the MDC

To record the values in the MDC, Logback uses a `ThreadLocal` variable.
This strategy works when one thread is used for one request, like in servlet container before the 3.1 specification.

Play framework, on the other hand, is [asynchronous](http://www.playframework.com/documentation/2.2.x/ScalaAsync). The processing of a request is composed of several function calls, and each call can be run on a different thread. ("Don't call me, I'll call you")

The implementation of the MDC with a `ThreadLocal` cannot work with this non-blocking asynchronous threading model.

## Defining a custom execution context

The dispatching of the jobs on different threads in done with an `ExecutionContext`. Each `ExecutionContext` manages a [thread pool](http://www.playframework.com/documentation/2.2.x/ThreadPools).

To use the MDC, we just have to use a custom `ExecutionContext` that propagates the MDC from the caller's thread to the callee's one.
```scala
import org.slf4j.MDC
import scala.concurrent.{ExecutionContextExecutor, ExecutionContext}

/**
 * slf4j provides a MDC [[http://logback.qos.ch/manual/mdc.html Mapped Diagnostic Context]]
 * based on a [[ThreadLocal]]. In an asynchronous environment, the callbacks can be called
 * in another thread, where the local thread variable does not exist anymore.
 *
 * This execution context fixes this problem:
 * it propagates the MDC from the caller's thread to the callee's one.
 */
object MDCHttpExecutionContext {

  /**
   * Create an MDCHttpExecutionContext with values from the current thread.
   */
  def fromThread(delegate: ExecutionContext): ExecutionContextExecutor =
    new MDCHttpExecutionContext(MDC.getCopyOfContextMap, delegate)

}

/**
 * Manages execution to ensure that the given MDC context are set correctly
 * in the current thread. Actual execution is performed by a delegate ExecutionContext.
 */
class MDCHttpExecutionContext(mdcContext: java.util.Map[_, _], delegate: ExecutionContext) extends ExecutionContextExecutor {
  def execute(runnable: Runnable) = delegate.execute(new Runnable {
    def run() {
      val oldMDCContext = MDC.getCopyOfContextMap
      setContextMap(mdcContext)
      try {
        runnable.run()
      } finally {
        setContextMap(oldMDCContext)
      }
    }
  })

  private[this] def setContextMap(context: java.util.Map[_, _]) {
    if (context == null) {
      MDC.clear()
    } else {
      MDC.setContextMap(context)
    }
  }

  def reportFailure(t: Throwable) = delegate.reportFailure(t)
}
```

Then we can define the default ExecutionContext in our application:
```scala
package concurrent

import scala.concurrent.ExecutionContext

/**
 * The standard [[play.api.libs.concurrent.Execution.defaultContext]] loses the MDC context.
 *
 * This custom [[ExecutionContext]] propagates the MDC context, so that the request
 * and the correlation IDs can be logged.
 */
object Execution {

  object Implicits {
    implicit def defaultContext: ExecutionContext = Execution.defaultContext
  }

  def defaultContext: ExecutionContext = MDCHttpExecutionContext.fromThread(play.api.libs.concurrent.Execution.defaultContext)

}
```

Now we will use the `concurrent.Execution.defaultContext` instead of the one from play (`play.api.libs.concurrent.Execution.defaultContext`)

## Using a custom execution context everywhere

Using a custom execution context is sometimes as easy as replacing
`import play.api.libs.concurrent.Execution.Implicits._` with `import concurrent.Execution.Implicits._`

The default [`Action`](http://www.playframework.com/documentation/2.2.x/ScalaActions) uses the default `play.api.libs.concurrent.Execution.defaultContext`.
We must define a custom `ActionBuilder` that uses our new `ExecutionContext`:
```scala
package controllers

object Action extends ActionBuilder[Request] {
  def invokeBlock[A](request: Request[A], block: (Request[A]) => Future[SimpleResult]) = {
    block(request)
  }

  /**
   * The standard [[play.api.mvc.Action]] loses the MDC context.
   *
   * This action builder sets the [[ExecutionContext]] so that the
   * MDC context is propagated.
   * With this custom [[ExecutionContext]], the request and the correlation IDs
   * can be logged.
   */
  override def executionContext: ExecutionContext = Execution.defaultContext
}
```

Instead of using of `play.api.mvc.Action`, we just have to use the newly defined `controllers.Action`.


With these customizations, we are now able to use the Mapped Diagnostic Context (MDC) with asynchronous actions written in Scala.

