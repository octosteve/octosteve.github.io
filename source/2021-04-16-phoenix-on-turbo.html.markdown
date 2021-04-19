---
title: Phoenix on Turbo
date: 2021-04-16 19:27 UTC
tags: phoenix, turbo, hotwire
---

Ok, so there's a lot of excitement around [Hotwire](https://hotwire.dev),
a way of building web apps by focusing on sending HTML Over the WIRE (get it!?).
Hotwire and the supporting library Turbo came out of building [hey.com](https://hey.com), an email service. It was
built to make Rails development snappier, and more dynamic. It also allows for an easy way to
build mobile apps backed by your webapp.

What's interesting is that it's not just limited to Rails. Since the whole idea is sending HTML
down to the client, we can use our favorite server side language to generate that markup
and join in on the fun.

Now, when I say "snappier and more dynamic" you say LiveView. I get it. In this post, we'll discuss
why this approach is a good way of getting high performance page rendering and better composibility
out of your existing request/response app.

<div>
  <img width="100%" src="https://images.unsplash.com/photo-1565052594002-8ee542c415bd?crop=entropy&cs=tinysrgb&fit=max&fm=jpg&ixid=MnwxMzc3OTd8MHwxfHNlYXJjaHwxfHxsaXZlJTIwd2lyZXxlbnwwfHx8fDE2MTgzNTQ5MjA&ixlib=rb-1.2.1&q=80&w=400" />
<small>Photo by <a href="https://unsplash.com/@cwrhoads?utm_source=seance&utm_medium=referral">Chris Rhoads</a> on <a href="https://unsplash.com/?utm_source=seance&utm_medium=referral">Unsplash</a></small>
<div>


## What is Turbo?
Turbo is made up of 3 components available in the `@hotwired/turbo` js library: Turbo Drive, Turbo Frames, and Turbo Streams.

### Turbo Drive
Turbo Drive gives your pages a nice boost for "free". It converts all of your http calls into XHR requests and replaces the body in line. If you think this sounds a lot like [turbolinks](https://github.com/turbolinks/turbolinks), you're right! Turbo Drive is the next generation of that concept.

You'll notice I used _free_ in quotes. Turbo Drive places state _in the browser_. From the [Handbook](https://turbo.hotwire.dev/handbook/building):
> Turbo is fast because it prevents the whole page from reloading when you follow a link or submit a form. Your application becomes a persistent, long-running process in the browser. This requires you to rethink the way you structure your JavaScript.

As Phoenix developers you'll notice one key difference between this and LiveView. Where state lives. BOTH are long lived processes and require us to change parts of how we build applications. With Turbo Drive, you can't rely on a page to load and setup your event handlers/socket connections etc. You've got to hook into the `turbo:load` event for setup and teardown. We'll see this in action when we talk about Turbo Streams over Channels.

The cool thing about Turbo Drive is that there's no code other than adding `import * as Turbo from "@hotwired/turbo"` into your `app.js` file.

### Turbo Frames
This is probably my favorite Turbo feature. FRAMES! Turbo
Frames let you compose web pages from _several_ endpoints,
letting you reuse your already existing markup on different
pages. This is huge! If you've got an existing HTML
endpoint, you can start to mix and match them using Turbo
by just marking up your... markup. Also, async requests are
a snap too. Just add an `src` attribute and it will it hit the
endpoint and load the results in-line.

### Turbo Streams
Turbo Streams allow you to send fragments of HTML via
WebSockets/Server Side Events. You'll receive a fragment of
HTML that, once you add it to the DOM, will position itself
relative to an anchor element and a directive. Sounds like
a lot, and to be honest, it takes some work to get it wired
up. While talking to the rest of my brain [Sophie
DeBenedetto](https://twitter.com/sm_debenedetto), she
called out that there's a lot of boilerplate to get this
working with Phoenix Channels vs just using LiveView. I
think one thing to keep in mind here is that Chris McCord
worked his butt off to make LiveView feel seamless. I'm
sure we can abstract this away to make it just as nice.

OK, let's build a thing.

## Issue Tracker
We're going to be adding Turbo to a simple issues app. It's
not tied to anything so really this is just a therapy app
where your friends can comment on your issues. You can pull
down the starter code
[here](https://github.com/octosteve/third_rail/tree/main).
As it stands, it's pretty unusable.

<div>
<img width="100%" src="https://i.imgur.com/S2Hzahe.gif" />
<div>

This is pretty standard scaffolding. You get to your first
wart when you create a new comment. Issues and Comments are
handled by 2 different controllers and require you to hit 2
different routes.

Let's add Turbo to see if we get any wins out of the box

### Adding Turbo
Run `npm install --save @hotwired/turbo --prefix assets`
at the root of your project. Then open up `app.js` and
include this.

<script src="https://gist.github.com/octosteve/ebf92fc9c1d77965376efc9b9a05d4ba.js"></script>

Let's see what we get.

<div>
<img width="100%" src="https://i.imgur.com/aW3Dpol.gif" />
<div>

Still terrible... but it's _fast_. You can see that all
requests to our backend were made via AJAX, with portions
of our page's `<body>` being replaced in-line.

Let's start moving towards making this... less terrible by
using Turbo Frames

## Showing Comments
As a thought experiment, let's think about _why_ you'd
want to have 2 separate endpoints for issues and comments.

1. If your authorization logic is complex enough, deciding which comments to render
and how people can interact with
them can become a pretty hairy bit of code. Keeping them
separate allows your endpoint to focus on doing one thing
well.
2. Speed. The data I need to render an issue is minimal.
Bringing in comments on that initial page load along with
pagination logic can slow the page down significantly.
3. None of these things are real for our stupid example,
but leave a comment on when you've seen this in the wild
and why you did it!

These reasons could be the best ones in the world, but no
one cares because it's unusable. Let's fix that.

Update the comments `index.html.eex` to
look like this:


<script src="https://gist.github.com/octosteve/8ef815eaa4423ce54980f6e12cbcb8b5.js"></script>

And replace update the Issue's `show.html.eex` template to
look like this.


<script src="https://gist.github.com/octosteve/55987ebabca10312bd81c8a0ff3c58a6.js"></script>


<div>
<img width="100%" src="https://i.imgur.com/yma3gOc.gif" />
<div>

THIS IS COOL! Think of those `<turbo-frame>` elements as
landing pads. Turbo uses those tags along with the `id` field
to pick out sections of the response and
replaces _just those portions in line_! Everything else is
ignored.

In our example, we're returning the _whole_ page back; layout and all when a user clicks `Show Comments` the link.
We can prevent the layout from being rendered by looking for the
the `turbo-frame` request header and only sending the relevant part of the page
back.

<script src="https://gist.github.com/octosteve/7d996d49ec823a605f09f5c0f762aaed.js"></script>

Now you get back _just_ the comments index, without the layout!



<div>
<img width="100%" src="https://i.imgur.com/FuTMMU9.png" />
<div>

This still kind of sucks though. What is this? the 90s? I
want my page to load, and I want a spinner! For this to
preload, all we need to do is change the issue's
`show.html.eex` to this.

<script src="https://gist.github.com/octosteve/40f47dc410bb13165e842c3b4c6b8d17.js"></script>

Let's see what this buys us, then we'll go over the code.
The takeaway from here is that we've _only_ added 5 lines
of markup, a plug to make it a bit quicker. The plug was optional!



<div>
<img width="100%" src="https://i.imgur.com/FFBbRL9.gif" />
<div>

I think we're starting to get _less_ terrible. GO TEAM.
Let's move on to Streams.

### Streams or Screams
So far, I've been cheating a bit. If you take a look at our
CommentController, I [redirect _back_ to the issue](https://github.com/octosteve/third_rail/blob/c86f23193777723ab6c1fc72a2017c67fef42d8f/lib/third_rail_web/controllers/comment_controller.ex#L15-L18). This
makes the page refresh, reload the comments and we're just about
ready to accept our UX awards. Streams present us another
opportunity to do something nice here. If we wrap our form in a `<turbo-frame`>, an accept header of
`text/vnd.turbo-stream.html` is added to the request. If we send back a specially
formatted response, along with a response content type, it integrates the response on the page.

Let's modify our controller to check for this header, and send
down a specially crafted message in response to a form
submission.

<script src="https://gist.github.com/octosteve/21865b2d93b411ecb57c711e5843be86.js"></script>

Following along with the numbers.

1. MORE PLUGS! Here we're going to added a value into
`conn.assigns`.
2.  If we can handle a stream for the response we...
3. add the `text/vnd.turbo-stream.html` content-type in the
response. THIS IS CRITICAL! Had to reach out the the
_great_ [Jamie Gaskins](https://twitter.com/jamie_gaskins)
for help on this bit. THANKS!
4. Finally we render a template that adds the magic bits
for the comment to wind up in the right place on the DOM.

We're going to modify the Comment's `index.html.eex` to return a new `turbo-frame` with an id of  `comment_list`.
This isolates _just_ the list of comments and excludes the header. We then add the fragment we'd like inserted
along with where we'd add it.

<script src="https://gist.github.com/octosteve/a195ab2e4ff955a6341d98e061d6cce5.js"></script>

From the [docs](https://turbo.hotwire.dev/handbook/streams)
you can see our `action` options are `append`, `prepend`, `replace`, `update`, and
`delete`. They do what you expect. The `target` is the id
of the element you're... well, targeting.

One last bit is you have to wrap the form in a
`<turbo-frame>`.


<script src="https://gist.github.com/octosteve/cb07d966ce06490dafdadbb414778978.js"></script>

Now we get this!



<div>
<img width="100%" src="https://i.imgur.com/r9rckHd.gif" />
<div>

Streams are a great feature, but we need to
smooth this out if we want any real usage.

There's one more bit of work we need to add to complete our
Turbo Tour. Updated comments over channels.

### Overview
Ok, so this is going to be a lot.

1. We're going to join a channel tied to an issue's comments.
1. When we visit a different issue, we'll need to disconnect from that channel and join a new channel.
1. We'll update the `create_comment_for_issue` function in our `Core` module to emit a `new_comment` message with the `id`.
1. In the channel, we'll look up the comment, and follow the Hotwire philosophy by sending down the HTML we want on the page.
1. Oh, and we'll have to convert the string of markup into DOM elements and add them to the body on the client side.

Like I said... a lot.


<div>
  <img width="100%" src="https://images.unsplash.com/photo-1521075486433-bf4052bb37bc?crop=entropy&cs=tinysrgb&fit=max&fm=jpg&ixid=MnwxMzc3OTd8MHwxfHNlYXJjaHwyfHxkZXNwYWlyfGVufDB8fHx8MTYxODU5NDc5Mw&ixlib=rb-1.2.1&q=80&w=400" />
<small>Photo by <a href="https://unsplash.com/@kaimantha?utm_source=seance&utm_medium=referral">Claudia Wolff</a> on <a href="https://unsplash.com/?utm_source=seance&utm_medium=referral">Unsplash</a></small>
<div>


### Setting up Channels

Channels are kind of magic in that unlike with LiveView, we don't need to manually subscribe via PubSub, and write our own `handle_info` function to update state. If we `broadcast` a message to a topic matching the channel name, unless we intervene, that message will be sent down to the client. To be clear, by joining the `comment:issue:9` channel, we will receive _all_ messages broadcast to the `comment:issue:9` topic.

Let's run `mix phx.gen.channel Comment`, and add
`channel "comment:issue:*", ThirdRailWeb.CommentChannel`
to `user_socket.ex`.

Let's update the channel to look like this:


<script src="https://gist.github.com/octosteve/65641a4cd92c4653ea2f90e368a42df6.js"></script>

We don't have to do anything with `issue_id` on join, but
that's where... security happens. We'll intercept the
`new_comment` message, and instead of just broadcasting the
`id` to the client, we'll convert it to the template we want on the page.
We need to intercept the message, otherwise only the `id` will be sent down to the client.

A quick glance at the broadcasting code:


<script src="https://gist.github.com/octosteve/18f895978f0857486f488fa99c97f4ba.js"></script>

Now onto the client!

### Setup and Teardown
Let's add a data element to the issue so we know which
channel to connect to. Open up the issues `show.html.eex`
file, and change the header to `<h1 id="issue" data-issue-id="<%= @issue.id %>">Show
Issue</h1>`. This will give us a way of knowing we're on a
page for a specific issue, _and_ which issue it is.

In the JavaScript, we'll

1. Listen for the `turbo:load` event
2. Disconnect from our channel, if we've already connected
3. See if we're on a page for an issue
4. Connect to the comments channel for that issue.
5. Respond to the `new_comment` message by appending it to
the DOM. Turbo does the rest.

<script src="https://gist.github.com/octosteve/0c610b5bcbf4e3438bd783647dbff300.js"></script>

We'll listen for that `turbo:load` event, and try to
disconnect. If the page we're on is an issue, we join that
channel. Here's `disconnectChannel` and `joinChannel` along with the message handler.


<script src="https://gist.github.com/octosteve/142378b7a7c083242485e56546e41447.js"></script>

Here we have the supporting code.`disconnectChannel` leaves the channel if it exists, then nullifies our variable (Yey! global mutable state!). `joinChannel` is where we conenct to that new channel and setup a listener for new messages.

`addToBody` uses a `DOMParser` instance to convert strings to HTML. Then then addes it to the bottom of the document body.

Let's see what that gets us.


<div>
<img width="100%" src="https://i.imgur.com/ZMkqlL5.gif" />
<div>

Yey? We're still inserting the `<turbo-stream>` response
from the form. We can get rid of that to remove the
duplicate comment. Revert it back to the previous version and
let's take a look.


<div>
<img width="100%" src="https://i.imgur.com/uhcaMYH.gif" />
<div>

Voilà! We're getting messages from a websocket and adding
them to the page! This also got us cross browser updates!


<div>
<img width="100%" src="https://i.imgur.com/I9DUPOV.gif" />
<div>

# Wrap
I really like how Turbo makes composing applications easy. I see
this as an analog to using LiveView Components to compose a larger View.
And the additional benefit of using your existing request/response endpoints is
compelling.

If you have a document based dashboard with some light interactivity, and
you've found yourself reaching for React, maybe give Turbo a try with some [Stimulus](https://stimulus.hotwire.dev/) sprinkled in.

Personally, I'd see having to add high interactivity to a page in Phoenix as the time to reach for LiveView.

Is Hotwire the future and is it worth adding it to your Phoenix apps? Leave a comment and let me know what you think.
