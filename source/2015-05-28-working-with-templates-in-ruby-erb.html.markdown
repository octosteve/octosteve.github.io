---
title: Working with Templates in Ruby ERB
date: 2015-05-28 15:39 UTC
tags:
---

# Templates in ERB
This post is a bit of a thought experiment. Looking into this led me down the route of some interesting ruby solutions.

## The problem
Ruby ships with the `erb` library, but it natively doesn't come with a way to wrap one ERB file in another. This makes composing files difficult. There's no way to use something like rails' `yield` method.

### Some Basic ERB

Here's how we'd render a simple erb template.

``` ruby
require 'erb'
template = "1 and 1 is <%= 1 + 1 %>"
ERB.new(template).result
```

This returns the interpreted string. If we try to use the `yield` keyword, we'll run into a bit of a problem.

``` ruby
require 'erb'
template = "<%= yield %>"
ERB.new(template).result #=> LocalJumpError: no block given (yield)
```

Eek! No bueno. You might be thinking, "Well, you have to give a block to... something". Good guess, but wrong.

``` ruby
require 'erb'
template = "<%= yield %>"
ERB.new(template).result do
  "HERE"
end
# or
ERB.new(template) do
  "HERE"
end.result
```

All sorts of crazy happens.

### Defining a Renderer
ERB does come with a curious set of methods that let you add the current context of the ERB instance (stuff like the template, variables, all that stuff), to another object.

That looks like this:

``` ruby
require 'erb'
# create a container for our new method
class LayoutRenderer
end

# load the erb instance
template = "Before <%= yield %> After"
erb = ERB.new(template)

# create method on LayoutRenderer. We'll pass a block to THIS method
erb.def_method(LayoutRenderer, 'render')

# Then call our LayoutRenderer
result = LayoutRenderer.new.render do
  "I'm somewhere in the middle"
end
result # => "Before I'm somewhere in the middle After"
```
Cool! Now it's a matter of rendering another template inside of it. We'll need a few files:

A layout:

``` html
  <!DOCTYPE html>
  <html lang="en">
  <head>
    <meta charset="UTF-8">
    <title>Document</title>
  </head>
  <body>
    <h1>Layout</h1>
    <%= yield %>
  </body>
  </html>
```
An index file:

``` html
<p>
  I'm in a template
</p>
```
And our renderer code:

``` ruby
require 'erb'

class LayoutRenderer
end

layout_template = File.read('layout.html.erb')
inner_template = File.read('index.html.erb')

layout = ERB.new(layout_template)
inner = ERB.new(inner_template)

layout.def_method(LayoutRenderer, 'render')

result = LayoutRenderer.new.render do
  inner.result # call the regular erb #result method
end

puts result
```

NOTE: If you're playing with this and try to `puts` this line...

``` ruby
puts LayoutRenderer.new.render do
  inner.result
end
```

... you'll get some weird issue. I think it's trying to finish the render before loading the block.

## Final Thoughts
This was fun to play with. Until next time, Happy Clacking.
