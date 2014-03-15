---
layout: post
title: "Server side rendering for JavaScript ReactJS framework"
date: 2014-03-15 09:44
comments: true
categories:
- Play Framework
- JavaScript
- ReactJS
---
## Flicker effect with JavaScript applications

A lot of web applications are nowadays build with a JavaScript framework, rendering the HTML in the browser (client side).
There are a few reasons for this, like:

- avoiding server-browser round-trips to modify one HTML element
- it is easier to keep the server side stateless if you maintain the state in the browser
- the server can expose a public REST API for partners. And your own JavaScript application can use this API, encouraging [eating our own dog food](http://en.wikipedia.org/wiki/Eating_your_own_dog_food)

Building a client side JavaScript application is not always easy for teams used to server side code, and a few frameworks can help there, like [AngularJS](http://angularjs.org/), [Ember](http://emberjs.com/) or [React](http://facebook.github.io/react/)

We will look at an [example with React](http://play-react.herokuapp.com/clientSide)

To display the HTML, a few steps are needed:

1. The browser loads HTML, CSS and JavaScript.<br>
   It can then display some HTML delivered directly by the server.<br>
	![The browser shows the HTML coming from the server](/assets/2014-03-15/server.png)

	With AngularJS, if [inline expression](http://docs.angularjs.org/guide/expression) are used, the user can see the following for a few milliseconds:<br/>
	hello \{\{firstname}}<br/>
	before AngularJS replaces this expression with its computed value.

2. The JavaScript framework manipulates the DOM and the user can then see the application.
	![The JavaScript application has changed the DOM](/assets/2014-03-15/server_and_client.png)


3. If the application needs to display some data from the server, it must first request it with Ajax. The data is displayed only after being received by the browser.
	![The JavaScript application has received data and changed the DOM accordingly](/assets/2014-03-15/server_and_client_and_data.png)

(to make the [flicker](http://play-react.herokuapp.com/clientSide) more visible, I introduced a latency of 500 ms to simulate a slow backend)

The user experience is not optimal. The application flickers at each step, as the DOM is changed several times in a few seconds.

## Avoiding the flicker effect
### On the client side

In the browser, we can mitigate the flicker effect.
Some applications show a spinner as long as the page is not ready to be shown.
The not-yet-completed DOM is hidden before being shown in one final step.

For example, AngularJS provides the [ng-cloak directive](http://docs.angularjs.org/api/ng/directive/ngCloak).
With this directive, AngularJS can hide the HTML as long as it is not ready.


### Welcome back to server side rendering

Another option is to pre-render the HTML on the server side and to serve a completely ready HTML to the browser.
That seems quite contradictory to the essence of JavaScript applications, but it is an interesting way.

Form example, React can render a UI component without any browser with [React.renderComponentToString](http://facebook.github.io/react/docs/top-level-api.html#react.rendercomponenttostring).

With this function, the complete page can be prepared on the server side, send under this form to the browser that can directly display the ready application. On the client side, the same JavaScript code can dynamically manipulate this DOM as a normal client side application.

The [React server rendering example](https://github.com/mhart/react-server-example) demonstrates how to use React's server rendering capabilities. Rendering a JavaScript application on the server side is technically possible because the JavaScript is executed by [Node.js](http://nodejs.org/).

### And what about the JVM?

If you are not using NodeJS, but the Java Virtual Machine (JVM), you might be disappointed at this time.
Pre-render a JavaScript application is only possible with Node.js?

In Java, there are a few projects that can save us:

- [trireme](https://github.com/apigee/trireme) provides a Node.js API and can run node.js scripts inside Java. It uses Rhino, the current JavaScript implementation for the JVM. (With Java 8, let's see if trireme will use the new JavaScript implementation, Nashorn, or whether Nashorn will implement the node.js API itself.)

- [js-engine](https://github.com/typesafehub/js-engine) provides [Akka Actors](http://akka.io/) to execute JavaScript code with trireme or with node.js

As a proof of concept, I implemented a little play application that uses these projects to pre-render a React component on the server side.

The JavaScript is loaded:
```scala
val serverside = Play.getFile("public/javascripts/serverside.js")
```

An actor is created for a JavaScript engine (trireme or node.js)
```scala
val engine = Akka.system.actorOf(jsEngine, s"engine-${request.id}")
```

We receive the data from the database:
```scala
data <- initialData
```
and let the JavaScript code execute with that data as parameter
```scala
result <- (engine ? Engine.ExecuteJs(new File(serverside.toURI), List(data))).mapTo[JsExecutionResult]
```
The result is send to the browser
```scala
Ok(views.html.index(Html(new String(result.output.toArray, "UTF-8"))))
```
complete controller code:
```scala
  // with js-engine
  def serverSideTrireme = serverSideWithJsEngine(Trireme.props())

  // with node
  def serverSideNode = serverSideWithJsEngine(Node.props())

  private def serverSideWithJsEngine(jsEngine: Props) = Action.async { request =>
    import akka.pattern.ask
    import scala.concurrent.duration._

    val serverside = Play.getFile("public/javascripts/serverside.js")
    implicit val timeout = Timeout(5.seconds)
    val engine = Akka.system.actorOf(jsEngine, s"engine-${request.id}")

    for {
      data <- initialData
      result <- (engine ? Engine.ExecuteJs(new File(serverside.toURI), List(data))).mapTo[JsExecutionResult]
    } yield {
      Ok(views.html.index(Html(new String(result.output.toArray, "UTF-8"))))
    }
  }
```

The code `serverside.js` uses the node.js API to render our main component (CommentBox).
```javascript
var React = require('./react'),
    CommentBox = require('./CommentBox');
```

It then loads the data given as first parameter in the controller
```javascript
// take data from parameters
var data = JSON.parse(process.argv[2]);
```

It renders the CommentBox component to a String and output it to console.log so that the Scala controller can receive the result with `result.output.toArray`

```javascript
console.log(React.renderComponentToString(CommentBox(backend)({data: data, onServerSide: true})));
```

Complete code:
```javascript
var React = require('./react'),
    CommentBox = require('./CommentBox');

// take data from parameters
var data = JSON.parse(process.argv[2]);

var backend = {
    loadCommentsFromServer: function(settings) {
    },
    handleCommentSubmit: function(settings) {
    }
};

console.log(React.renderComponentToString(CommentBox(backend)({data: data, onServerSide: true})));
```

[This page](http://play-react.herokuapp.com/serverSide) does not flicker anymore compared to the [first version](http://play-react.herokuapp.com/clientSide).


### Drawback with server side rendering

The drawback with pre-rendering the page on the server side is that we have to wait to have all the data before sending the page.
In the [sample application](http://play-react.herokuapp.com/serverSide), I introduced a latency when requesting the data to simulate a slow database.

The browser must also wait long before getting any HTML. The following diagram shows that the application deployed on Heroku delivers the page in more than 1s!
![The browser is waiting for the server](/assets/2014-03-15/wait_for_server.png)


### Can we optimize more?

We can optimize this version by sending the first bytes of the HTML page before having any data.
When the data is there, we can send the rest of the page.

With [that variant](http://play-react.herokuapp.com/serverSideStream), we can include the CSS and part of the JavaScript in the \<HEAD\> section of the HTML page.
The browser receives this information very quickly and can begin downloading these assets.
The server lets the connection open and when the rest of the page is ready, it is send to the browser.

![The browser can load the CSS and JavaScript](/assets/2014-03-15/browser_loads_assets.png)

To implement this, I used the Facebook’s BigPipe concept as presented in the [talk “Building composable, streaming, testable Play apps” from Yevgeniy Brikman](http://de.slideshare.net/brikis98/composable-and-streamable-play-apps)

It is not a "Silver Bullet" as we are still waiting for the data before displaying it to the user (that makes sense).
But the browser can load the stylesheets, the JavaScripts very quickly, leading to a more responsive page.

If you need more information, the [code is available on github](https://github.com/yanns/play-react).