---
layout: post
title: "play framework meetup in Berlin in November 2015"
date: 2015-11-17 21:43:55 +0100
comments: true
categories: 
---

During the [play framework meetup in Berlin in November 2015](http://www.meetup.com/Play-Berlin-Brandenburg/events/226561633/):

## for a better workflow between designers and developers

[Laura](https://twitter.com/lluizesc) made a very good talk about how to better organize the work between front-end designers and developers.

For that, her team build some tools:

- the designer uses an assets pipeline in nodejs to build a website. The templates are written with [handlebars](http://handlebarsjs.com/). When the website is ready, a release is made and all assets are packaged as a [webjar](http://www.webjars.org/): https://github.com/lauraluiz/handlebars-webjars-demo

- the developer uses a play framework application that is capable of using directly the assets released by the designer: https://github.com/lauraluiz/play-handlebars-demo

Slides: http://slides.com/lauraluiz/handlebarsplay

Project using this workflow: https://github.com/sphereio/sphere-sunrise


## type classes in Scala


I introduced type classes in Scala: http://www.slideshare.net/yann_s/introduction-to-type-classes-in-scala

Step by step, I explained how the Json Reads/Writes/Formats work in the [scala API of the play framework](https://www.playframework.com/documentation/2.4.x/ScalaJson).

