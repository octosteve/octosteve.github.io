---
title: Processes in Elixir
date: 2016-04-26 15:47 UTC
tags: [Elixir, Processes, Concurrency]
---

# Processes
> The unit of concurrency in Elixir is a process. A process is  capable of responding to messages, and maintaining state. The only way to interact with a process is to send it a message.

>  **Steven Nunez (Hostile Developer)**

## Creating a process
We'll be climbing the ladder of abstraction but let's start at the bottom.

![getinsertpic.com](http://media0.giphy.com/media/byvw3EmW6L37q/200.gif)

We use the `spawn` function to create a new process. We pass it a function and that code will run in an isolated process.

```elixir
pid = spawn(fn ->
  IO.puts("I run in a different process)")
  IO.inspect(self)
end)
```

This code will print its process id. Don't be fooled! It's running in a separate process, but `IO.puts` is set to print output to the `group_leader`. That's your main console by default.

Try checking if this process is alive and you'll be met with sad news:
```elixir
Process.alive?(pid) # => false
```
Once a process' is done with it's job... it dies. Cruel world.
![getinsertpic.com](http://media3.giphy.com/media/XrT2XN8L6yoMg/200.gif)

To interact with the process, we have to teach a process how to receive a message.
```elixir
pid = spawn(fn ->
  receive do
    {:help} -> IO.puts "Helping!"
  end
end)
Process.alive?(pid) # => true
```

Doing this will keep the process alive. We can send this process a message to trigger the corresponding code.

```elixir
send(pid, {:help}) # prints Helping!
Process.alive?(pid) # => false
```
### send and receive
`send` is how we send messages to a process. We can send ANY message to a process, and it's up to the process to respond to the message. You define which messages you respond to with a `receive` block and a list of messages you can respond to.

One consideration here: *If you don't give your process a way to respond to a message, it will keep  it unprocessed. This could lead to memory issues by filling your MAILBOX (real term, look it up)*

Pretty easy to solve for this. Modify our last `receive` block to look like this:

```elixir
pid = spawn(fn ->
  receive do
    {:help} -> IO.puts "Helping!"
    _ -> IO.puts "Don't know what to do with that"
  end
end)
send(pid, :beef) # prints "Don't know what to do with that"
```
Once something matches code in the `receive` block, the block exits. If that's the last thing in the process, the process dies. We'll need to define a loop to create an immortal process.

## loops with recursion

For this next part, we'll define a module with a loop function.
```elixir
defmodule Help do
  def loop do
    receive do
      {:help} -> IO.puts "Helping!"
      _ -> IO.puts "Don't know what to do with that"
    end
    loop # <------ Super important!
  end
end
```

Then we'll run this function in a process using `MFA` notation.
```elixir
pid = spawn(Help, :loop, [])
send(pid, {:help}) # prints "Helping!"
send(pid, {:beef}) # prints "Don't know what to do with that"
send(pid, {:help}) # prints "Helping!"
```

What's going on? First off, in our `loop` function, we're doing the same `receive` block, preventing our process from moving forward. Once something matches, we call `loop` again, locking us in a receive block again.

When working with modules, you can spawn processes in a couple of ways. We used `MFA` which stands for Module/Function/Arguments, but we could have done it this way.

```elixir
spawn(&Help.loop/0)
```

Both work fine.
## Return to sender

Until now, we've been printing our result to the screen, but that's returning a value to **you** dear reader, not your process. To do that, we'll need to have our `loop` function send a message instead of printing. For THAT to work, we'll need to include the sender in the message we send. Referencing the current process is easy: use `self`.

```elixir
defmodule Help do
  def loop do
    receive do
      {:help, help_seeker} ->
        send(help_seeker, {:reponse, "Helping!"})
      _ ->
        IO.puts "Don't know what to do with that"
    end
    loop
  end
end
helper = spawn(Help, :loop, [])
send(helper, {:help, self})
# There's a message in my mailbox!!!!
flush
```

Wall of code! We updated our `receive` to match on the tuple `{:help, help_seeker}`. We later send this process a message, passing the console process (yes, it's a process too!) as the one seeking help. We finish up by calling `flush` to show our messages in the console.

# Wrap up
We've seen 2 patterns for processes. Tasks where you care about the return value, and others where you don't. There a time and  a place for both, and in the next post, we'll see how Elixir provides an abstraction for handling these messages with GenServers.
