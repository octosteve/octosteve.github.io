---
title: Hey! Watch it!...or how to monitor files in Elixir
date: 2016-02-24 16:57 UTC
tags: [elixir, file watchers, fs]
---

## File watchers and Elixir
I'm working on a feature that reloads a config file if the file is updated. The easiest way I could figure out was to watch the file, and reload the code. Thinking in JavaScript, I'd expect something like:

``` javascript
// fake
Dir.watch('configs/', function(changedFile){
  console.log(`${changedFile} was updated`);
})
```

But if you've been working with Elixir for a bit, you'll know there's gonna be a whole bunch of message passing to get this working. In the end, we wind up with something like:

``` elixir
:fs.start_link(:my_watcher, Path.absname("config"))
:fs.subscribe(:my_watcher)
receive do
  {_watcher_process, {:fs, :file_event}, {changedFile, _type}} ->
   IO.puts("#{changedFile} was updated")
end
```

## Getting Started and the FS library

I picked the `fs` library since it's what Phoenix uses. In this post, we'll be using [master ](https://github.com/synrc/fs/tree/812449edbf4c756c54ed2373e1c7353fe74c036a) in a mix project. Follow along!

### A new project!

Create a new mix project with `mix new watch_it`, and add fs to your dependencies in mix.exs.

```elixir
#...
defp deps do
  [{:fs, github: "synrc/fs"}]
end
#...
```

And start the `fs` application with:

```elixir
def application do
  [applications: [:logger, :fs]]
end
```

Then install everything with `mix deps.get`

And kick the tires in `iex -S mix` (this will compile a bunch).

``` elixir
:fs.start_link(:my_watcher, Path.absname("config"))
:fs.subscribe(:my_watcher)
flush
# {#PID<0.129.0>, {:fs, :file_event},
# {'/Users/StevenNunez/code/watch_it/config/fake_config.exs', [:created]}}
```

**I don't know about you, but I'm excited.** What happened?
`:fs.start_link(:my_watcher, Path.absname("config"))` created a process that watches a directory for changes. We give the process a reference of `:my_watcher`. We'll use it later. The next line is where the magic is.
`:fs.subscribe(:my_watcher)` finds the `:my_watcher` process, then notifies `self` whenever a file changes.

We can add a bit of pattern matching to make this nicer. (Make sure you're exiting `iex -S mix` between examples)

``` elixir
:fs.start_link(:my_watcher, Path.absname("config"))
:fs.subscribe(:my_watcher)
receive do
  {_watcher_process, {:fs, :file_event}, {changedFile, _type}} ->
   IO.puts("#{changedFile} was updated")
end
```

### Conclusion

I plan on using this library a lot! Using this with separate process responding to changes to the file system is super powerful. Phoenix uses it for it's live reloading. Maybe we can replace some JavaScript build tools ;-)
