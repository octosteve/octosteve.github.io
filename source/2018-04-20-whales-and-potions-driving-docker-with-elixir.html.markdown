---
title: Whales and Potions. Driving Docker with Elixir
date: 2018-04-20 22:44 UTC
tags: Elixir, Docker, Unix Sockets
---

[On an episode of SoBroCodeBros](https://www.youtube.com/watch?v=q44qxXxsIUQ), we explored connecting to Docker's API with Elixir. The way the show works is my brother and I pick a topic we're pretty good at, and use it in an unfamilar domain. In this case, I'm really familar with Elixir and Docker, but working with it through it's Unix Socket interface was new.

The intention of this post is to summarize some of our learnings. As always, be sure to [Like and subscribe](https://www.youtube.com/channel/UC0tLqS7es8MmM_rrcTF1cbA) to our channel over on YouTube, and let me know if you want to see this blog post in video format. Off we go.

## Assumptions
You're running at least Elixir 1.6, and have Docker installed. I'm running API version 1.37 but I don't think we're using any bleeding edge docker stuff.

## Getting Started

Let's create a plain Elixir project with `mix new docker_driver`, then open up `mix.exs`. Here we'll add a few dependencies. `HTTPoison` to make requests, and `Poison` to parse JSON responses.

``` elixir
# mix.exs
defp deps do
  [
    {:poison, "~> 3.1"},
    {:httpoison, "~> 1.0"},
  ]
end
```

and run `mix deps.get` 

Then create a new module to wrap a few of our operations named `docker_adapter.ex` in `lib/docker_driver/`.

## Connecting to Unix Sockets

Here's our first pass at our module.

``` elixir
defmodule DockerDriver.DockerAdapter do
  @socket  "/var/run/docker.sock" |> URI.encode_www_form
  @url     "http+unix://#{@socket}"

  def running_containers do
    with {:ok, %HTTPoison.Response{body: body}} <- HTTPoison.get("#{@url}/containers/json"),
         {:ok, parsed} <- Poison.decode(body) 
    do
      parsed
    else
      err -> 
        IO.puts "something Broke"
        IO.inspect err
    end
  end
end
```

Add this to `.iex.exs` in your project so you don't have to keep aliasing stuff

``` elixir
alias DockerDriver.DockerAdapter
```

Start a container and backround it. That way something will show up in our shell in a bit with `docker run -itd busybox`.

In `iex -S mix` run `DockerAdapter.running_containers`

```
[
  %{
    "Command" => "sh",
    "Created" => 1524266416,
    "HostConfig" => %{"NetworkMode" => "default"},
    "Id" => "3ccab1ee2ee3509cf8b955604638cca4ed2044b0f71434876d490837158defda",
    "Image" => "busybox",
    "ImageID" => "sha256:8ac48589692a53a9b8c2d1ceaa6b402665aa7fe667ba51ccc03002300856d8c7",
    "Labels" => %{},
    "Mounts" => [],
    "Names" => ["/affectionate_pare"],
    "NetworkSettings" => %{
      "Networks" => %{
        "bridge" => %{
          "Aliases" => nil,
          "DriverOpts" => nil,
          "EndpointID" => "558ee96cab03aafaf33c1233422fdf2601eca404473d2abec8f4593fe706ceb0",
          "Gateway" => "172.17.0.1",
          "GlobalIPv6Address" => "",
          "GlobalIPv6PrefixLen" => 0,
          "IPAMConfig" => nil,
          "IPAddress" => "172.17.0.2",
          "IPPrefixLen" => 16,
          "IPv6Gateway" => "",
          "Links" => nil,
          "MacAddress" => "02:42:ac:11:00:02",
          "NetworkID" => "2993a0d3c34044b0f615eb8c149342d5ce9148835258260e66fca1366b6ac473"
        }
      }
    },
    "Ports" => [],
    "State" => "running",
    "Status" => "Up 10 seconds"
  }
]

```
DATA


link to https://docs.docker.com/engine/api/v1.30/#operation/ContainerList, do Inspect, then logs, followed by attaching to a stream.
