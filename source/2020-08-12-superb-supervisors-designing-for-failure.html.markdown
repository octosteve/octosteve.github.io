---
title: Superb Supervisors. Designing for Failure
date: 2020-08-12 04:19 UTC
tags: elixir, supervisors, failure
---

Supervisors are "have you tried turning it off and on
again?" turned into a programming strategy. We've all made
that trade off. Should I keep debugging what's wrong, or
just reboot? Often, the reboot fixes it and we can just
move on.

But supervisors can also be used to design systems that go
down and stay down. In this post we're going to talk about
when we'd want to design this kind of system and how
exactly to do it.

## The Problem
We're going to design a system of RabbitMQ consumers that
fail at the first sign of trouble. Do not pass go, do not
collect 200 dollars.
<div> <img src="https://images.unsplash.com/photo-1544393569-eb1568319eef?ixlib=rb-1.2.1&q=80&fm=jpg&crop=entropy&cs=tinysrgb&w=400&fit=max&ixid=eyJhcHBfaWQiOjEzNzc5N30" /> <div> Photo by <a href="https://unsplash.com/@nadineshaabana?utm_source=seance&utm_medium=referral">Nadine Shaabana</a> on <a href="https://unsplash.com/?utm_source=seance&utm_medium=referral">Unsplash</a>

You might be asking yourself, "Why would you build a
crashing consumer?" These consumers listen on the wire to
"Facts". These Facts are non-negotiable. If you opt in for
a message, you MUST integrate it. Failure to do so
indicates you have issues with your underlying data. Your
read models may be out of sync for instance. Continuing to
consume message, even on DIFFERENT queues may lead to
your system doing its job poorly. For instance, if you are
consuming messages about an online store's inventory,
prices, available discounts, and sales for an app that
tries to run projections of monthly sales, you'd better
alert someone if an inventory update fails to get consumed.
Here's what we'll be building:
<div class='mermaid'>

graph TD
Application --> ConsumerGroup
ConsumerGroup --> ConsumerSupervisor
ConsumerSupervisor --> MushroomConsumer
ConsumerSupervisor --> ToxicityConsumer
ConsumerGroup --> ConsumerMonitor
ConsumerMonitor -.-MushroomConsumer
ConsumerMonitor -.-ToxicityConsumer
</div>

Let's talk about what's going on here. Our application
starts a ConsumerGroup. This is a Supervisor that starts 2
complementary processes, a ConsumerSupervisor, responsible
for starting our consumers, and a ConsumerMonitor. We want
our consumer monitor to... Monitor Consumers. At the first
sign of danger, it will instruct the ConsumerSupervisor to
stop the presses, and kill all of it's children.
<div> <img src="https://i.imgur.com/RpagL6h.png" /> <div>

After we've fixed the problem, we'll bring everything back
online. OK, let's get started.

## Some Homework
Since we covered how to setup a Producer and Consumer Pools in
a [previous
post](https://hostiledeveloper.com/2020/08/09/connection-pools-and-rabbitmq.html)
I won't go into too much detail here. After setting them
up, our Supervison tree should look like this.
<div> <img src="https://i.imgur.com/CUPLoFj.png" /> <div>

Great, on to our supervisors.
## Bottom Up
Let's start at the base of our Supervision tree, the
Consumers. We're going to use ExRabbitPool's Consumer
module to save us some boilerplate, and we'll customize our
restart strategy to support our "Burn the world" approach.
<script src="https://gist.github.com/StevenNunez/9d2f70621f8b780d05b51a9a1b9c78c5.js"></script>

The biggest difference here is we set the restart option
to `:temporary`. Supervised processes can set 1 of 3
restart option.
1. `:permanent` (default): If it dies, bring it back to
life no matter what.
2. `:transient`: Only bring me back if I die under
suspicious conditions. If I die with a "normal" reason,
then it's fine. Get me nice flowers.
3. `:temporary`: If I die at all, leave me dead.
My friends, what are we if not temporary processes, trying
to handle the right messages, lest we be killed by one
destined for someone else?

`:temporary` works for us here since we want to stop
consumers from potentially doing more damage. If it dies,
let it die.

The `ToxicityConsumer` looks pretty similar, except it has
a different exchange and queue.
<script src="https://gist.github.com/StevenNunez/3b810cae5094d909d3227c17cefc5144.js"></script>

On to the `ConsumerSupervisor`. Since it's supervising
processes that have the `:temporary` restart option, the
strategy doesn't really matter. We're going to leave it
with the default `:one_for_one` strategy.
<script src="https://gist.github.com/StevenNunez/b20d803d25979e1944bef6a76fd834ff.js"></script>

We've added a couple of additional functions that we need
to support our `ConsumerMonitor` process. The first one is
a list of all of the pid's this supervisor is managing. The
second function terminates all of the children.
<div> <img src="https://i.imgur.com/fL89Fqv.png" /> <div>

The `ConsumerMonitor` will do well by its namesake but take
a look at its restart option.
<script src="https://gist.github.com/StevenNunez/5ee793e6b707f601041ccdb787e69a72.js"></script>

We're setting it's restart strategy to `:transient`. Reason
being, if this puppy dies for ANY other reason than what's
on line 21, I want it alive. Notice we pass in a supervisor
as the argument to `start_link/1`.

On start, we monitor each of the supervisor's pids and
just wait... as soon as a process dies, we instruct the
supervisor to execute order 66.
<div> <img src="https://i.imgur.com/6lGqb8n.png" /> <div>

The `ConsumerGroupSupervisor` ties it all together. Pay
special attention to the strategy option.
<script src="https://gist.github.com/StevenNunez/1317e3e7d12dde6e9e698ed4e751f3ca.js"></script>

Supervisors get started sequentially. We __completely__ start
the `ConsumerSupervisor` before we start the
`ConsumerMonitor`. The ensures the pids are started and
ready to be monitored. The `:rest_for_one` strategy allows
the monitor to fail and recover without disturbing the
consumers, but will allow us to _heal_ the system. More on
that later.

Let's add this to our application and take it out for a
spin.
<script src="https://gist.github.com/StevenNunez/1e9b41eab6b66ee867cd1569a0e862e1.js"></script>

Let's take a look at what this looks like in observer.
<div> <img src="https://i.imgur.com/pe27V20.gif" /> <div>

## Supervisors are awesome
Our tree dies when it's supposed, and comes back up in a fresh state when needed. LOVE IT!
<blockquote class="twitter-tweet"><p lang="en" dir="ltr">That feeling when you configure the perfect supervision tree ❤️ <a href="https://twitter.com/hashtag/myelixirstatus?src=hash&amp;ref_src=twsrc%5Etfw">#myelixirstatus</a> come see what I&#39;m talking about over at <a href="https://t.co/zaKaXqCwt5">https://t.co/zaKaXqCwt5</a></p>&mdash; Steven Nunez 🇩🇴🇺🇸🙅 (@_StevenNunez) <a href="https://twitter.com/_StevenNunez/status/1292215852469825539?ref_src=twsrc%5Etfw">August 8, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script> 
