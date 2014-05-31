---
layout: post
title: "enlarge your test scope"
date: 2014-05-30 22:00:22 +0200
comments: true
categories:
- Play Framework
---
At the beginning at the year, I had the chance to present [how to organize a play application with the Cake pattern](blog/2014/02/17/ping-conf-2014/) at [Ping Conf](http://www.ping-conf.com/).
I showed how this pattern enable designing the application as components, how to reduce visibility of specific gateway's model only to components that need it. One side-effect of the cake pattern is that it allows a dependency injection resolved at compilation time.

In one of my last slides, I warned against abusing a dependency injection mechanism to write too much unit tests.
To stay within the time slot, I have not developed so much my concerns about that point.

{% img http://image.slidesharecdn.com/play-with-cake-export2-140121150250-phpapp01/95/slide-66-638.jpg %}

During the talk, I implemented an application to demonstrate my points. This application consumes two external web services to render HTML pages. It is quite a typical application we can see in an infrastructure build with micro-services.

I've now took the time to write a new version of this application I used in the demo.
[And This new version is not using any unit tests but only some sort of component tests.](https://github.com/yanns/TPA/tree/master/frontend/TBA_07)

Let's dig into the details how this new version differs from the ones build around the Cake pattern.

### Traditional view of unit tests

When building an application, we usually structure the code into layers to separate responsibilities, thus enabling re-use of logic, and avoiding repetition.

In the demo I used for the talk, the application is for example layered into views, controllers, services and gateways. All these layers have access to a common model.

{% img http://image.slidesharecdn.com/play-with-cake-export2-140121150250-phpapp01/95/slide-10-638.jpg" 'code structured in layers' %}

A traditional approach of unit test is to consider one class or function of one layer as a unit to test. The other dependent layers are mocked.

For example, to test the service layers, we use mocks for the gateways, simulating responses from external web services.

{% img http://image.slidesharecdn.com/play-with-cake-export2-140121150250-phpapp01/95/slide-21-638.jpg %}

This approach works well, but has several downsides:

- the unit tests prove that one layer is working as expected, but they said nothing about all the layers used together.
- the unit tests use the internal details of the implementation. Re-factoring the code implies then to change a lot of tests.

By using dependency injection and mocks, it is nowadays very easy to write unit tests. The effect if some applications are tested almost only with unit tests:

{% img http://image.slidesharecdn.com/play-with-cake-export2-140121150250-phpapp01/95/slide-65-638.jpg %}

### Traditional view of component tests

To complement the unit tests, a team can decide to implement some component tests.

For the sample used in the talk, the component is the font end application we are currently implementing.

The most common way to run component tests is to start the tested application. The application is configured not to use the real external web services, but some local mock implementations. The local mock implementations are started as http servers as well, running on different ports.

{% img http://image.slidesharecdn.com/play-with-cake-export2-140121150250-phpapp01/95/slide-13-638.jpg %}

When the application and all the mocks are started, we can test the application by sending some http requests and by analyzing the responses.

Setting up the test environment with this approach is quite complex. For each external web service, a mock must be implemented as a real local http server. We must start all mock implementations, and then the new configured application. At the end of the tests, we must shutdown all services, even in case of exceptions.

But the main drawback with this approach in my opinion is that running the tests take a lot of time, too much to be integrated in a normal development flow (write code -> compile -> test)


### An alternative approach between component and unit tests

To strictly adhere to the definition of component tests, we should treat the tested application as a black box, and simulate all external web services. We saw that this approach is somewhat heavy to use: each external web service must be mock with a real http server.

Starting and running the tests in that configuration take time. Debugging an error can be difficult.

The strategy I used in new version of the demo application ([TBA_07](https://github.com/yanns/TPA/tree/master/frontend/TBA_07)) is a little bit different.
The idea is still to use a request / response to test the application, but without having to run the application and any external web services.

Implementing that is actually quite easy: each layer declared as dependency an HTTP client (a [WSClient](http://www.playframework.com/documentation/2.3.x/api/scala/index.html#play.api.libs.ws.WSClient) in play framework 2.3)

- The http client is a dependency at the top (controllers' layer):
```scala
package controllers

class Application(ws: WSClient, app: play.api.Application) extends Controller {

  val topVideoService = new TopVideoService(ws, app)
//...

}
```
(a second "dependency" is the current play application. This approach is very convenient to simulate different configurations)

- The real implementation of the http client is then "injected" at the last time, when we construct the controller singleton:
```
def playCurrent = play.api.Play.current
object Application extends Application(WS.client(playCurrent), playCurrent)
```

- To test the application, we then simply have to instantiate the controller with an [alternative implementation of the http client capable of simulating external web services](https://github.com/yanns/TPA/blob/master/frontend/TBA_07/test/httpclient/MockWS.scala):
```scala
class ApplicationControllerFixture
  extends Application(MockWS(playerRoute), FakeApplication())
```

- The `playerRoute` simulate the external player web service:
```scala
val playerId = PlayerId(34)

val playerRoute: MockWS.Routes = {
  case ("GET", u) if u == s"http://localhost:9001/players/$playerId" =>
    Action { Ok(Json.parse(playerJson(playerId))) }

  case _ => Action { NotFound }
}

def playerJson(playerId: PlayerId) =
  s"""{
       |  "id": $playerId,
       |  "name": "ze name",
       |  "height": "ze height",
       |  "weight": "ze weight",
       |  "team": "ze team"
       |}
     """.stripMargin
```

The `MockWS.Routes` type defines a partial function `PartialFunction[(String, String), EssentialAction]`, making really easy to combine different routes together with `orElse`:
```scala
SimulatedVideoBackend.videoRoute orElse SimulatedPlayerBackend.playerRoute
```

- and we can test the response by calling the controller with a [`FakeRequest`](http://www.playframework.com/documentation/2.3.x/api/scala/index.html#play.api.test.FakeRequest):
```scala
val result = index.apply(FakeRequest())
status(result) mustEqual OK
```

The application is constructed as if it was depending from the http client and the current play application.

These tests are not strictly component tests, as we are not testing the real implementation of the http client.
The application is not entirely treated as a black box. But most of the code is tested like in production.


#### Drawbacks of this approach:
- writing a test is more complicated than testing a little unit of code
- writing unit test can help avoiding code mixing responsibilities. We do not profit from that.
- when a test suddenly fails, it is more difficult to find out why.
- we do not test the complete application stack. For example, the [play filters](http://www.playframework.com/documentation/2.3.x/ScalaHttpFilters) and the real http client is not tested.


#### Advantages of this approach:
- a developer must understand how the application works in general to be able to write good tests
- the application can be re-factored without modifying the tests
- the user functions are better tested
- the integration of all layers together is tested
- we do not need any running server to check all the tests. The tests run very fast.
- the code is simple (compare the [TopVideoService in that version](https://github.com/yanns/TPA/blob/master/frontend/TBA_07/app/services/TopVideoService.scala) with the [one using the Cake pattern](https://github.com/yanns/TPA/blob/master/frontend/TBA_05_final/app/services/TopVideoServiceComp.scala))

### Experience with that approach

With one team, we are currently testing this approach. The results are quite encouraging: more than 80 % of the statements are covered by tests. We have more than 200 tests running in 10 seconds on my machine.

And I could deeply change the code with almost no impact on the tests. ;)
