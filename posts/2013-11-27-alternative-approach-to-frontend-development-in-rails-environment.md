---
title: "Alternative approach to frontend development in Rails environment"
created_at: 2013-11-27 13:56:33 +0100
kind: article
publish: false
author: anonymous
tags: [ 'foo', 'bar', 'baz' ]
---

Alternative approach to frontend development in Rails environment

### Typical approach

Let's take a look at vastly oversimplified example.

__app/assets/javascripts/greeter.coffee__:

    class Greeter
      greet: =>
        alert('hi!')

    window.Greeter = Greeter

__app/assets/javascripts/home.coffee__:

    #= require greeter

    g = new Greeter()
    g.greet()

Notice the line window.Greeter = Greeter. It's needed in order to use Greeter class in home.coffee. Typically one exports all classes to window.
Most of the time that works pretty fine. When you have lots of CoffeScript code, especially when developing a Single Page Application, you notice that it's a distinct app from the backend, that just happen to be deployed by Rails.

### Using Brunch

I will show you now how to decouple your frontend from your backend using neat little tool called brunch.

[Setting up brunch]

__frontend/app/home.coffee__

    Greeter = require('greeter')

    g = new Greeter()
    g.greet()

__frontend/app/greeter.coffee__

  class Greeter
    greet: =>
      alert('hi!')

  module.exports = Greeter

__index.html__

    <script>require('home');</script>

I know, this looks awfully similiar to what we had before. But let's take a look at the file Brunch produced:


__app.js__

    # little bit of Brunch code
    require.register("home", function(exports, require, module) {
      var Greeter, g;
      Greeter = require('greeter');
      g = new Greeter();
      g.greet();
    });


Notice that compiled content of `home.coffee` is wrapped in a function.
When you load application.js generated by Assets Pipeline, all the code gets executed immediately. This is not the case with JS generated by brunch. It won't run before you explicitly use require. This gives you control over when each part of the code is used.

The second difference is in scope. Notice that when you do `window.Greeter = Greeter`, you pollute global scope. Code in one file can use any class that happen to be exported to window previously. However `module.exports = Greeter` only registers it inside the module. In this approach you don't create any global values and you have to require all the dependencies explicitly by require. In a big codebase it can really improve maintainability.

Now your frontend is completely decoupled from Rails. If you deal with CORS, it can be even deployed on different server.