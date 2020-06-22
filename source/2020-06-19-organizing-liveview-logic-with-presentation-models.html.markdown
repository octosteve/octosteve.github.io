---
title: "Organizing LiveView Logic with Presentation Models"
date: 2020-06-19 17:31 UTC
tags: Elixir, Phoenix, LiveView, Design Patterns
---

I've been really enjoying Phoenix LiveView! Working on smart interfaces is super easy and just works. It's really incredible.

I've been going through the [PragmaticStudio LiveView course](https://pragmaticstudio.com/phoenix-liveview) and have been LOVING IT. If you just hearing about it, SIGN UP NOW (well maybe after reading this post...). You'll thank me. It's really well done and is the best way to get your head around the awesomeness of LiveView.

They build a series of UI patterns in LiveView to show you how simple it is to add seemingly complex interactions into a Phoenix app. In one module, the implement search. Having been a teacher in a previous life I know that sometimes you have to keep things simple at the cost of implementing better patterns. One such example is [this LiveView](https://github.com/pragmaticstudio/live_view_studio/blob/0420ed745e71f931d17d70e32f5be1b72cfa1495/lib/live_view_studio_web/live/search_live.ex).

There's nothing _wrong_ with their code. It works! But I think it suffers from tutorial-itis, where it's enough to prove a point, but not what you would actually ship.

## Thinking in Layers
LiveViews suffer from the same neck snapping, eye darting issues as GenServers. With a GenServer you define a client API, then a server API. Check the client API for the arguments passed in. Did we pattern match on the client side? Look for `handle_*` to see what it does, and how it handles errors. Also, testing complex logic in GenServers is less than great. But that's fine! We have a way to deal with this by decoupling our [Boundary Layer](https://pragprog.com/titles/jgotp/) from our core business logic, breaking them into nice, pure functions we can easily test.

LiveViews are a bit different though. The data we want to isolate isn't really part of our _core_ business logic. In fact, in most cases, it exists to deliver a single experience in your app.

Let's take a look at the code example. It stores a few pieces of state that get manipulated over the course of the page's life.
<script src="https://gist.github.com/StevenNunez/f384871008825b16bc09819267f28721.js"></script>
In the example, the `LiveViewStudio.Stores` module runs our search and is very much part of our _core_ business logic. This might be some code that gets optimized and used in several places in our app in a number of contexts. The results from our search will be stored in `stores`. Our LiveView can potentially act on the data it got back from the data store, further decorating it in preparation to render it. Along with `zip` and `loading`, this specific interpretation of `stores` would never be used outside of this view. Why does this matter?

## Separation of concerns and testing
The more complex our transformations get, the more state we have to hold at different points of this page's life in response to user actions or other process' messages, the more we'd benefit from separating out a functional core.

Now I know that LiveView has [great ways](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveViewTest.html) of mimicking clicks, and changes from your front-end, as well as a way to send messages to that process to simulate internal messages, but we still want to keep our Boundary Layer small, and extract our logic into small testable modules.

A quick note before we move on... I know this is going to be a LOT of code for solving a relatively small problem. I think this pattern will come in handy when you're dealing with apps that take advantage of `live_action` and have a single LiveView render several subcomponents. For now... Go with me.

## Extracting the Module
[Presentation Model](https://martinfowler.com/eaaDev/PresentationModel.html)
> Represent the state and behavior of the presentation independently of the GUI controls used in the interface - Martin Fowler

We're going to extract the state for this component, and encapsulate the logic a bit. We showed the state, but what's this thing do?
Users are presented a search box, where, the request takes 2 seconds to complete. Between submission and bringing up results, a `loading` icon appears.

<iframe src="https://giphy.com/embed/fA1JrSOgltr5VoPvGK" width="480" height="352" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p><a href="https://giphy.com/gifs/fA1JrSOgltr5VoPvGK">via GIPHY</a></p>

The flow looks something like:
1. Send query to LiveView
2. LiveView sets a loading state, sends self message to actually RUN the query, and returns the new `loading` state.
3. Query finishes asynchronously, updates state with results, clears the loading state and returns that to the view.

Let's start out with some tests:

<script src="https://gist.github.com/StevenNunez/03c6e76849653104ec97f2b777f0e67b.js"></script>
Our module expects to receive a function that takes one argument. Note we've broken up the flow between preparing the query (storing it) and actually executing it.
OK, on to the code.

<script src="https://gist.github.com/StevenNunez/c347f9339e48142c922b994ce0991b31.js"></script>
OK, so this is a lot of code, compared to what the final result I linked to earlier but let's think about what we've won. There are clearly 3 things this module can do. Each of those things has a clear pipeline describing what it means to execute it. And best of all, it's all pure! We don't have to simulate clicks, or mess with input fields. We can test plain old values. Adding a new feature to this and verifying it works with automated tests is trivial. Imagine how you would add the ability to ensure a zip code is only numbers? Or that they're 5 characters long?

## Updating the LiveView
We're going to take a look at what the `SearchLive` LiveView would look like using this Presentation Model.



<script src="https://gist.github.com/StevenNunez/0e7db950bfcfd93743cdbe1199042b60.js"></script>
SO NICE! Our live view is solely responsible for making sure the right parts of our Presentation Model module get called and use it to render state. Notice how we're using `@presentation_model.x`. If you're wondering, there's no hit to performance using this! LiveView does a diff on the server before sending the change over the wire. If there's no change, it doesn't send anything. And what's best is that to test the LiveView, you only have to test a couple of cases for success and failure, as opposed to every detail and edge case of the view logic.
This is kind of great.

## Wrap up
LiveView is awesome, and with the right patterns we can ensure our fancy UIs have equally fancy tests. You can build really complex logic that you will KNOW works. Thanks for reading. Happy Clacking.

