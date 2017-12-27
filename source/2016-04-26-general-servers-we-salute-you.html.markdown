---
title: General Servers we salute you
date: 2016-04-26 22:15 UTC
tags: elixir
published: false
---

# Atteeeeeention!

General Servers, or `GenServers` abstract a bunch of the business of process management from us, and is probably the first bit of OTP you'll get to work with. OTP is massive. Just know that it's a set of really common patterns abstracted out by the Erlang neckbeards in the sky. We Salute you!

![getinsertpic.com ](http://media3.giphy.com/media/l3V0q1JDkTaUzpgm4/200.gif)

## Why Use GenServers
We saw in the [last post](/2016/04/26/processes-in-elixir.html) that Elixir offers us some primitives for sending messages between processes, specifically `send` and `receive`. In this post we're going to get a brief intro into GenServers, an abstraction for server processes.

### Working with GenServers
We can buy in to working with GenServers by using the GenServer *behaviour* (#england).

```elixir
defmodule Help do
  use GenServer
end

{:ok, pid} = GenServer.start_link(Help, [], [])
```

Introducing the world's most boring piece of software. This is a simple GenServer. One thing to note: We create a new process using `GenServer.start_link/3`, not spawn. I'm going to skip over what those empty lists are for a moment. Trust me for a second.

Arguments and what they mean,
Defining your own start link
Defining init

Process communication, working well with supervisors, define syncronous and asyncronous messages.

handle_cast
handle_call
handle_info
![getinsertpic.com](http://media1.giphy.com/media/zwhemlyzWIYlq/200.gif)
