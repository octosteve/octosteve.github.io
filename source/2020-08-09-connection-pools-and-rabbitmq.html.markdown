---
title: Connection Pools and RabbitMQ
date: 2020-08-09 02:04 UTC
tags: elixir, rabbitmq
---

In this post, we're going to go over connection pools how to use ex\_rabbit\_pool to manage connections and channels.
These concepts will be covered further in my [course at ElixirConf](https://2020.elixirconf.com/trainers/4/course).

## Connection Pools

If you've used Ecto, you've used connection pools. Connection pools allow you to allocate a number of connections to a service and allow processes in your system to use those connections to do work concurrently. In the case of Ecto, when you make a query, one of the available connections is checked out, used to make the query and return results. When it's done, it goes back in the pool.

With RabbitMQ ideally you're going to have [1 connection per resource](https://www.cloudamqp.com/blog/2018-01-19-part4-rabbitmq-13-common-errors.html). One connection for publishing, and one connection for consuming. When we use connection pools with Rabbit, we're _really_ going to be configuring a **channel pool**.

## Connections and Channels
Think of a connection as a tunnel, and a channel as a bunch of runners that go through the tunnel.
![](https://unsplash.com/photos/3fzYRVQu_3c/download?force=true&w=640)

Photo by Marcos Nieto on Unsplash

These slackers aren't very inspiring.

Channels allow you to multiplex over a single TCP connection. Since connections are pretty heavy (costing the server about 100KB), channels give us a way find the resource sweet spot. The [Channels guide](https://www.rabbitmq.com/channels.html#resource-usage) notes that pooling should be used when managing your own channels becomes a pain but why not go for broke from the jump?

## Getting Started
Let's start with a project:

<script src="https://gist.github.com/StevenNunez/6d1a15f7a45572c9a39cbcf94e33cdae.js"></script>
Open up `mix.exs` and add [ex\_rabbit\_pool](https://github.com/esl/ex_rabbit_pool), then run `mix deps.get`

<script src="https://gist.github.com/StevenNunez/82cb9836bc4a7d1648cf46e62509341f.js"></script>
Great! Let's start out with the easier of the 2. The publishing pool. Again, we're going for a SINGLE connection but multiple channels. Since publishers don't block, we can keep the count pretty low. Let's start out with 2 channels.

<script src="https://gist.github.com/StevenNunez/fcf9706108409911248a133ebd6a9320.js"></script>
A whole-lotta code. Let's jump right in:

1. If you've worked with [Poolboy](https://github.com/devinus/poolboy) before, this syntax should look familiar.
We're configuring 1 `ExRabbitPool.Worker.RabbitConnection` process to be checkout-able. These configs go straight to Poolboy.
2. These rabbit configs get passed in to the worker when it's checked out.
3. You've been using these sweet `child_spec/1` functions since Elixir 1.5! They let you define what it means to _start_ your server.
Here, we're doing a little... but with a lot of code. `id` is what your Supervisor will use to track your process. `start`, well, that's how you start the dang thing. Notice we're just delegating to `ExRabbitPool.PoolSupervisor.start_link/2`. That second argument to the Pool Supervisor's `start_link` is the name you can use to reference this process.

Add this to `application.ex`
<script src="https://gist.github.com/StevenNunez/8318539fd3d5e82cd1a7ca97b1f41436.js"></script>
Run `iex -S mix` and you'll see this in observer.

<a href="https://imgur.com/cnWlTDU"><img width="100%" src="https://i.imgur.com/cnWlTDU.png" title="source: imgur.com" /></a>

Open the Rabbit Console and you'll see 1 connection and 2 channels. Just what we want!

<a href="https://imgur.com/tPwPhop"><img src="https://i.imgur.com/tPwPhop.png" title="source: imgur.com" /></a>

Let's create a publisher the publishes to a `what_should_we_do` exchange with a payload of "Pool Time!"
<script src="https://gist.github.com/StevenNunez/f19d7a058ae5350feb0e936383f8f584.js"></script>

The `ExRabbitPool.RabbitMQ` module [delegates all of it's calls](https://github.com/esl/ex_rabbit_pool/blob/master/lib/clients/rabbitmq.ex) to the underlying
[amqp](https://github.com/pma/amqp) library, but we'll use this to support any
changes they may make in the future. Don't worry about declaring the exchange multiple
times. The only time this will cause a problem is if you call it with different configurations
on subsequent calls.

In `iex` run `Pooler.PoolTimePublisher.publish`, then open the rabbit console to see
your new exchange!

<a href="https://imgur.com/iczQAPS"><img src="https://i.imgur.com/iczQAPS.png" title="source: imgur.com" /></a>


## Consumers
Peanut butter has Jelly, Publishers have consumers. We're going to take a different approach to consumers, instead using a provided macro that hides all of the
fetching and returning stuff the `ExRabbitPool.with_channel/2` function was doing for you.

First, we need a new pool.

### Consumer Pool
Very similar to the Publisher Pool, except we're going to bump the channels to 5. This is something we'd tweak over time, with a good signal being that your consumers occasionally crash because they can't get a channel. This could be due to consumers taking a long time to finish a request, or having more concurrent work than you have channels. In any event, this is something you'll have to tweak for your environment.

<script src="https://gist.github.com/StevenNunez/ce73de390dc416f17e22ba2bf258a7ae.js"></script>
Same as before except with a higher channel count. Be sure to update `application.ex`

<script src="https://gist.github.com/StevenNunez/a999e4089dc034d27786adc104ba19b5.js"></script>
On to our Consumer.
The best APIs offer nice abstractions but let you reach into the internals when needed. We'll see how in a bit.
We're going to create a queue that consumes messages from
our exchange. We're going to use a named durable queue to allow us to stop consumers
and not lose messages.

<script src="https://gist.github.com/StevenNunez/816d77c74b0850c985fd76a8ae5de357.js"></script>

A lot going is on here but let's take a look.

1. We're defining our `child_spec/1` that passes in the consumer pool we defined and a queue.
2. `setup_channel/2` is one of the defined hooks that let us setup any additional connections and bindings prior to consuming messages on the queue. As previously mentioned, don't worry about redeclaring the exchange, or the queue for that matter. You'll only get an issue if you try to change its properties.
3. This is where the money is 💰💰💰. `basic_deliver/3` gets called when a message makes its way to our queue. We're on the hook for returning `:ok` or `{:stop, reason}` to kill the server.

The other callbacks defined are `basic_consume_ok/2`, triggered when a consumer successfully joins, and
kbasic_cancel/3` `basic_cancel_ok/2` for handling cancellation events.

Don't forget to update `application.ex`

<script src="https://gist.github.com/StevenNunez/7a29ab01e5986873c732168518713368.js"></script>

## Test Drive
Let's see how you work with this thing. Let's publish a message that says "Pool Time!" on to the `what_should_we_do` exchange.

Start firing off those messages with `Pooler.PoolTimePublisher.publish/0`.

<a href="https://imgur.com/OhIVclm"><img src="https://i.imgur.com/OhIVclm.png" title="source: imgur.com" /></a>

SUCH POWER!!

## Why bother
This might seem like a big lift, but there's one big benefit to not having to manage your own channel
handshakes. Failure. If you were managing channels and connections on your own, you'd have to write a slew of other processes to
free up resources when you failed. Or you'd have to set up separate monitors to clean up after your GenServers. With
`ExRabbitPool` you keep things in a clean, expected state. Let's update our consumer to raise an exception like an
underslept teenager. We're going to change the restart strategy to `:temporary` to not bring down the world,.

<script src="https://gist.github.com/StevenNunez/fa6109cd145231b5b980fac2e8da89db.js"></script>

Go ahead and publish some messages. Sure it crashes, but look at your stats.

Rabbit is holding on to your message until you get can get your act together.

<a href="https://imgur.com/EnLDYSG"><img src="https://i.imgur.com/EnLDYSG.png" title="source: imgur.com" /></a>

... and your channels are ready to field new requests.

<a href="https://imgur.com/xIjvVdl"><img src="https://i.imgur.com/xIjvVdl.png" title="source: imgur.com" /></a>

That's all for today. Thanks for reading.
