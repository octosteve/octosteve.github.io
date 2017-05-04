---
title: Something Useless - Terminal in your browser
date: 2017-05-02 13:40 UTC
tags: elixir, something useless
---

# Terminal in your browser

In today's installment of Something Useless, we're going to build a terminal in your browser with Phoenix!
This post will be broken up into 3 sections.

* Setting up Phoenix and Channels
* Creating a GenServer to act as a proxy for our Shell
* Tying it all together

## Getting Started
Create a new Phoenix Application. We'll be using Phoenix 1.3.

``` shell
mix phx.new terminal_in_your_browser --no-ecto
# Install all your dependencies
cd terminal_in_your_browser
```

Let's start by getting the terminal on the page. Start by adding the `xterm` package.

``` shell
cd assets
npm install --save xterm
cd ..
```

Then edit `brunch-config.js` to load the styles for the plugin. Without this... it's just a `textarea`.

``` javascript
// brunch-config.js
// ...
npm: {
 enabled: true,
 styles: {
   "xterm": ["dist/xterm.css"],
 }
}
// ...
```

Clear out `lib/terminal_in_your_browser/web/templates/page/index.html.eex` and insert a `div` with an id of `terminal-container`.


```html
<div id="terminal-container"></div>
```

Finally, in `assets/js/app.js ` load `xterm` and load it on the page:

```javascript
import Terminal from 'xterm'

let term = new Terminal({
  cursorBlink: true,
})

term.open(document.getElementById('terminal-container'))
term.on('data', (data) => console.log(data))
```

That last bit will console.log every character you type. Off to a great start!

## Flipping Channels
Now we get to work with some Channels!

Let's create the Terminal Channel. We'll use to receive input from our `xterm` box in the browser.

```shell
mix phx.gen.channel Terminal
```

Follow the instructions and but instead, add this to `user_socket.ex`

```elixir
channel "terminal:*", TerminalInYourBrowser.Web.TerminalChannel
```

Let's clean up the `TerminalChannel` module to only include a `join/3` function allows anyone to join.

```elixir
defmodule TerminalInYourBrowser.Web.TerminalChannel do
  use TerminalInYourBrowser.Web, :channel

  def join("terminal:" <> id, payload, socket) do
    {:ok, socket}
  end
end
```

The fun part. Let's echo everything you type back to you from the channel.

In the channel add this:

```elixir
def handle_in("input", %{"input" => input}, socket) do
  push(socket, "output", %{output: input})
  {:noreply, socket}
end
```

We push the input back to the client with an event named `output`. Our current code doesn't know how to handle this but it will soon. The `{:noreply, socket}` just tells phoenix not to reply to the sender, and preserves the socket to this process.

Next up. let's edit the `socket.js` file Phoenix provides us:

```javascript
import {Socket} from "phoenix"                  

let socket = new Socket("/socket", {})

socket.connect()                                

export default socket
```

Initialize a socket, connect, and export. Super simple. We'll use this socket in `app.js`. Add this near the top of the file.

```javascript
import socket from './socket'                   

let channel = socket.channel("terminal:1", {})
channel.join()
```
With this, we're connected to our server! Refresh the page and checkout the logs.

```shell
[info] JOIN "terminal:1" to TerminalInYourBrowser.Web.TerminalChannel
  Transport:  Phoenix.Transports.WebSocket
  Parameters: %{}
[info] Replied terminal:1 :ok
```

NICE! Let's wrap this leg of the trip up by sending all terminal input to our channel, then respond to the `output` event we send from the channel.

In the end, your `app.js` should look like this:
```javascript
import Terminal from 'xterm'
import socket from './socket'

let channel = socket.channel("terminal:1", {})
channel.join()
channel.on('output', ({output}) => term.write(output)) // From the Channel

let term = new Terminal({
  cursorBlink: true,
})


term.open(document.getElementById('terminal-container'))
term.on('data', (data) => channel.push('input', {input: data})) // To the Channel
```
Lines where we send IO back and forth are marked in comments. Look! Our terminal... just returns what we typed. Not super exciting. Up next, we'll be delving into the bowels of Unix, and learning how to interact with external processes.

## Ports, ptys, AKA why did I choose to write about this?!

Erlang comes with a built in way to communicate with external programs called `Ports`. With a port you can spawn a process like a shell, or a ruby script and communicate via IO. I can send it messages with `Port.command/2`, and receive formatted responses in the calling process. This has some limitations for our purposes. We need a terminal that sends back an echo when we type. As you saw, our terminal doesn't actually put anything in the box UNLESS we send it. We _could_ write this ourselves but why. \*nix has us covered. If you dig a bit deeper, you figure out that the reason it doesn't echo your input is because it's not a PTY. Let's look at an example. In `iex` type the following:

``` elixir
iex(1)> port = Port.open({:spawn, "$SHELL"}, [])
#Port<0.1355>
iex(2)> Port.command(port, "ls\n")              
true
iex(3)> flush()
{#Port<0.1355>,
 {:data, 'assets\n_build\nconfig\ndeps\nlib\nmix.exs\nmix.lock\npriv\nREADME.md\ntest\n'}}
:ok
iex(4)> Port.command(port, "l")   
true
iex(5)> flush()
# no echo
:ok
```

It might be hard to see, but when we flush on `3` we get back the response we see the port sent us a message. On `5`, no joy, no echo which we need to make our terminal work. Also, we don't get our lovely prompt.

The thing that causes the terminal to echo are the settings of the `stty` command. You can see your current settings by typing in `stty --all`. Mine look like this:

```shell
$ stty --all
speed 38400 baud; rows 23; columns 96; line = 0;
intr = ^C; quit = ^\; erase = ^?; kill = ^U; eof = ^D; eol = M-^?; eol2 = M-^?; swtch = M-^?;
start = ^Q; stop = ^S; susp = ^Z; rprnt = ^R; werase = ^W; lnext = ^V; discard = ^O;
min = 1; time = 0;
-parenb -parodd -cmspar cs8 hupcl -cstopb cread -clocal -crtscts
-ignbrk brkint -ignpar -parmrk -inpck -istrip -inlcr -igncr icrnl ixon -ixoff -iuclc ixany
imaxbel iutf8
opost -olcuc -ocrnl onlcr -onocr -onlret -ofill -ofdel nl0 cr0 tab0 bs0 vt0 ff0
isig icanon iexten echo echoe echok -echonl -noflsh -xcase -tostop -echoprt echoctl echoke
-flusho -extproc
```
If you run `stty -echo` you'll see it turns your echo off! Sadly, running `stty echo` on our port doesn't make it echo. Instead, we get a weird error:

```shell
iex(6)> Port.command(port, "stty echo\n")
true
iex(7)> stty: 'standard input': Inappropriate ioctl for device
```

Womp. This error is essentially telling us our port isn't a pty. To get around this, we're going to bring in a library that 'provides significantly better control over OS processes than built-in `erlang:open_port/3` command'. It also has pty support. The library we're talking about is `erlexec`.

## erlexec
With `erlexec`, we'll be able to get around this pesky pty issue, and turn on our terminal echo. Let's get it setup.

In our `mix.exs`, let's add the latest version of `erlexec`. Your deps should look like this.

```elixir
defp deps do
  [{:phoenix, "~> 1.3.0-rc"},
   {:phoenix_pubsub, "~> 1.0"},
   {:phoenix_html, "~> 2.6"},
   {:phoenix_live_reload, "~> 1.0", only: :dev},
   {:gettext, "~> 0.11"},
   {:cowboy, "~> 1.0"},
   {:erlexec, "~> 1.7"},
  ]
end
```

If you're running elixir >= 1.4, you won't need to start the application. If you aren't, make sure you add `:erlexec` to your application list.

Run `mix deps.get`, then run `iex -S mix` to try it out.

``` elixir
iex(1)> {:ok, pid, _os_pid} = :exec.run('$SHELL', [:stdin, :stdout, :stderr, :pty])
iex(2)> :exec.send(pid, "l")                                                    
iex(3)> flush()
{:stdout, 10721,
 "\e]0;stevennunez@TashiDor: ~/code/blog_code/terminal_in_your_browser\astevennunez@TashiDor:~/code/blog_code/terminal_in_your_browser$ "}

 ```

We start our default shell with the `:stdin` option so it accepts input from use via `:exec.send/2`. `:stdout` and `:stderr` make it so the calling process receives output in a tagged tuple starting with `:stdout` and `:stderr`.
Notice that the only message we've received so far is our terminal prompt. Also note that we're not getting an echo from the `l` we sent in.

Next, we'll finish up the `ls\n` command by passing in the rest.

``` elixir
iex(4)> :exec.send(pid, "s\n")
iex(5)> flush()
{:stdout, 10721,
 "\e[0m\e[01;34massets\e[0m  \e[01;34m_build\e[0m  \e[01;34mconfig\e[0m  \e[01;34mdeps\e[0m  \e[01;34mlib\e[0m  mix.exs  mix.lock  \e[01;34mpriv\e[0m  README.md  \e[01;34mtest\e[0m\r\n"}
{:stdout, 10721,
 "\e]0;stevennunez@TashiDor: ~/code/blog_code/terminal_in_your_browser\astevennunez@TashiDor:~/code/blog_code/terminal_in_your_browser$ "}
```

Here we get a file list. The wacky `\e[om\e` characters are for formatting in the terminal program. All `xterm.js` compatible!

Let's try turning on the echo via `stty` settings.

``` elixir
iex(6)> :exec.send(pid, "stty echo\n")
iex(7)> :exec.send(pid, "l")          
iex(8)> :exec.send(pid, "s")
iex(9)> flush()
{:stdout, 10721,
 "\e]0;stevennunez@TashiDor: ~/code/blog_code/terminal_in_your_browser\astevennunez@TashiDor:~/code/blog_code/terminal_in_your_browser$ "}
{:stdout, 10721, "l"}
{:stdout, 10721, "s"}

```

ECHO!

A couple of things to note:
1. `{:ok, pid, _os_pid} = :exec.run('$SHELL', [:stdin, :stdout, :stderr, :pty])`. Note the single quotes. This __has__ to be an Erlang String or Elixir Charlist, which is in single quotes.
2. `:exec.send(pid, "ls\n")` Double quotes when you `:exec.send`

## GenServer time
Let's wrap all of this in a `GenServer`. The server will have a way to receive input as well as a way to notify a process (our socket process) when there's something to output.

Step 1: Make a module named `Terminal` that uses `GenServer`

```elixir
defmodule Terminal do
  use GenServer

end
```

Step 2: Create `start_link/1` and the `init/1` callback to receive and store the output_pid:

```elixir
defmodule Terminal do
  use GenServer

  def start_link(output_pid) do
    GenServer.start_link(__MODULE__, output_pid)
  end

  def init(output_pid) do
    {:ok, %{output_pid: output_pid}}
  end
end
```

Step 3: In the `init/1` callback, start a pty shell and set it up to echo on input:

```elixir
defmodule Terminal do
  use GenServer
  #... start_link omitted
  def init(output_pid) do
    {:ok, shell, _os_pid} = :exec.run('$SHELL', [:stdin, :stdout, :stderr, :pty])
    :exec.send(shell, "stty echo\n")

    {:ok, %{
      output_pid: output_pid,
      shell: shell
      }
    }
  end
end
```

Step 4: Take input and send it to the shell pid:

```elixir
defmodule Terminal do
  use GenServer
  #... previous code omitted

  def send_input(terminal, input) do
    GenServer.cast(terminal, {:input, input})
  end

  def handle_cast({:input, input}, %{shell: shell} = state) do
    :exec.send(shell, input)
    {:noreply, state}
  end
end
```

Step 5: Handle output your `GenServer` receives as a message:

```elixir
defmodule Terminal do
  use GenServer
  #... previous code omitted

  def handle_info({:stdout, _os_pid, output}, %{output_pid: output_pid} = state) do
    send(output_pid, {:output, output})
    {:noreply, state}
  end

  def handle_info({:stderr, _os_pir, output}, %{output_pid: output_pid} = state) do
    send(output_pid, {:output, output})
    {:noreply, state}
  end
end
```

And that's it for our `GenServer`! You'll notice we send a message to our `output_pid`. How do we handle that? Let's jump back to the socket.

## TerminalChannel part 2

Let's dive in.
``` elixir
defmodule TerminalInYourBrowser.Web.TerminalChannel do
  use TerminalInYourBrowser.Web, :channel

  def join("terminal:" <> id, payload, socket) do
    {:ok, terminal} = Terminal.start_link(self())
    {:ok, assign(socket, :terminal, terminal)}
  end

  def handle_in("input", %{"input" => input}, socket) do
    Terminal.send_input(socket.assigns[:terminal], input)
    {:noreply, socket}
u end

  def handle_info({:output, output}, socket) do
    push(socket, "output", %{output: output})
    {:noreply, socket}
  end
end
```

When you join, we create a new terminal process. We pass in `self()` as the `output_pid`. We then add the terminal pid to the socket's state. Then we wait... When we get that `input` message from the browser terminal, we send it to the terminal process. Here's the cool part:

__Channels are just `GenServer`s so they respond to out of band messages with `handle_info`.__
 Whoop!

And just like that, we've got a terminal in our browser!

## Wrap up
The `erlexec` library is really powerful. With options to send `stdout|stderr` to different processes, the way we set this up could have been really different.

If there's anything you want to hear more about, leave it in the comments below. I'm always looking for ways to contribute.

Thanks!
