---
layout: post
title: "migrate a playframework app from 2.3 to 2.5"
date: 2016-04-26 09:31:45 +0200
comments: true
categories: 
---

Recently I migrated several [play](https://www.playframework.com/) applications from the version 2.3 to the version 2.5.

As this experience may interest other people, I decided to write about it.

So let's start!

# [§](#migrate-from-2.3-to-2.4) <a name="migrate-from-2.3-to-2.4"></a> Migration from 2.3 to 2.4

First, we will migrate the application from the version 2.3 to the version 2.4.

For that, the [migration guide](https://www.playframework.com/documentation/latest/Migration24) will be our reference.

How do I proceed?

### [1.](#update-sbt-config) <a name="update-sbt-config"></a> Update the sbt config to play 2.4

First, I update the SBT configuration:

- update the play version in `project/plugins.sbt`:
```
-addSbtPlugin("com.typesafe.play" % "sbt-plugin" % "2.3.10")
+addSbtPlugin("com.typesafe.play" % "sbt-plugin" % "2.4.6")
```

- if needed, update the sbt version in `project/build.properties`:
```
sbt.version=0.13.11
```

- update the `build.sbt`:
```
-import PlayKeys._
+import play.sbt.PlayImport._
```
I also add to replace `playVersion.value` with `play.core.PlayVersion.current`

The goal of this first step is to be able to load the application with `sbt`, even if the application itself does not compile. At this point, Intellij can load the application with a play 2.4 version, and can provide auto-completion.


### [2.](#fix-compilation) <a name="fix-compilation"></a> Fix compilation errors

Then, I fix the compilation

Inside sbt, I just trigger the compilation:
```
~compile
```
and fix all compilation errors. The play team has made a very good job at documenting all steps in the [migration guide](https://www.playframework.com/documentation/latest/Migration24).

At this point, I just want the application to compile with the minimal number of changes. I do not care if I use deprecated APIs.


### [3.](#run-application) <a name="run-application"></a> Run the application

When all compilation errors are fixed, I just make sure that the application can run:

In sbt:
```
~run
```

At this point, I sometimes discovered version conflicts between libs (ex: Netty versions) and I know if I can complete the migration or if I have to update a dependency before.

Analysing all dependencies can be difficult.
I use the [sbt-dependency-graph](https://github.com/jrudolph/sbt-dependency-graph) to find overriding versions.


### [4.](#fix-tests) <a name="fix-tests"></a> Fix the tests

Then I fix all compilation errors in the tests, and then check that the tests are successful.

In sbt:
```
~testQuick
```

When all the tests are green, the application is migrated.


### [5.](#remove-deprecated-usages) <a name="remove-deprecated-usages"></a> Remove all usages of deprecated API

Now it is time to clean our application and to fix all code using a deprecated API.

I use the Scala API, and I prefer not to use any runtime dependency injection framework.

I introduced the so-called [_compile time dependency injection_](https://www.playframework.com/documentation/2.5.x/ScalaCompileTimeDependencyInjection) to simply instantiate my controllers in one application loader.

For more info about this topic, please refer to:

- [my talk about it](http://de.slideshare.net/yann_s/play-24dimacwire) at the [play meetup](/blog/2015/05/20/di-with-play-2-dot-4/)
- [this great post from Loïc Descotte](http://loicdescotte.github.io/posts/play24-compile-time-di/)


# [§](#migrate-from-2.4-to-2.5) <a name="migrate-from-2.4-to-2.5"></a> Migration from 2.4 to 2.5

For that, the [migration guide](https://www.playframework.com/documentation/latest/Migration25) will be our reference.

same steps as before:

- [update the sbt config](#update-sbt-config), but to play 2.5:
```
-addSbtPlugin("com.typesafe.play" % "sbt-plugin" % "2.4.6")
+addSbtPlugin("com.typesafe.play" % "sbt-plugin" % "2.5.2")
```
- [fix the compilation errors](#fix-compilation), like in application loader:
```
-    Logger.configure(context.environment)
+    LoggerConfigurator(context.environment.classLoader).foreach { _.configure(context.environment) }
```
- [run the application](#run-application)
- [fix the tests](#fix-tests)
- [clean up the application](#remove-deprecated-usages)<br>
For example, I can remove the `InjectedRoutesGenerator` as it is now the default.
```
-routesGenerator := InjectedRoutesGenerator
```
