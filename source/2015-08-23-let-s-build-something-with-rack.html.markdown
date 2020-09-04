---
title: Let's build something with Rack
date: 2015-08-23 20:28 UTC
tags: rack, ruby
---

# Slow down!

When it's time to build a web app, we go for the big guns. `gem 'rails'` in the `Gemfile` and we're off to races. What if we didn't? What if we tried to carefully craft every line in our app? What if we only added [Sprockets](https://github.com/sstephenson/sprockets) when we needed it? What if we want granular control of our [app's security](https://github.com/hassox/warden)? Or what if we just want a better idea of what's actually happening when a user visits our site?

## Rack - The Bow Drill of Web Apps
Let's build a simple web app using just rack and rack middleware. Along the way we'll learn about some rack features like URLMap and writing custom middleware.

## The App - Programmer of the Day
What are we without our past? Let's write an app where we can keep track of our programming heroes. The app, PotD is our one stop for learning about awesome coders.


### Planning
We'll need a few routes:

* GET / gives you a link to the "All Programmers" page
* GET /programmers returns a list of our programmers
* GET /programmers/4 returns a programmer with an id of 4

We'll also add Bootstrap since all websites need Bootstrap.

### Starter App
I've set up a database and basic setup [here](https://github.com/octosteve/rack_no_rails). Other than `rack` being in the `Gemfile` there's nothing webby about this!


## Rackup!
In our `config.ru` file we'll start off by handling the `GET /` requirement.

```ruby
class App
  def call(env)
    request = Rack::Request.new(env)
    if request.get? && request.path == "/"
      Rack::Response.new("You're at the home page!")
    else
      Rack::Response.new("File not found", 404)
    end
  end
end

use Rack::ContentType
run App.new
```

If this looks crazy, take a look at the my [Rack Basics](/2015/08/21/lets-learn-about-rack.html) post.

Running `rackup` starts the app. Let's make the homepage load a template.

```ruby
class App
  def call(env)
    request = Rack::Request.new(env)
    if request.get? && request.path == "/"
      template = File.read('app/views/home/index.html.erb')
      Rack::Response.new(template)
    else
      Rack::Response.new("File not found", 404)
    end
  end
end

use Rack::ContentType
run App.new
```
Put this template in app/views/home/index.html.erb. We're not actually using erb for this template, but let's keep the template names consistent.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Potd</title>
</head>
<body>
  <h1>Welcome to Programmer of the Day!</h1>
  <ul>
    <li>
      <a href="/programmers">Click here</a> to get a list of our programmers.
    </li>
    <li>
      <a href="/about">Click here</a> to learn more about the app.
    </li>
  </ul>
</body>
</html>
```
If you're thinking about `yield` and templates and all that... go read [this](/2015/05/28/working-with-templates-in-ruby-erb.html). I'm not touching it in the post.

Cool! One story DONE! But this code looks crazy already. Let's break some of this code into controllers.

```ruby
# HomeController in app/controllers
class HomeController
  def index
    template = File.read('app/views/home/index.html.erb')
    Rack::Response.new(template)
  end
end
```
In `config.ru` we bring in our `environment` file so it loads all our classes.

```ruby
# config.ru
require_relative 'config/environment'
class App
  def call(env)
    request = Rack::Request.new(env)
    if request.get? && request.path == "/"
      HomeController.new.index
    else
      Rack::Response.new("File not found", 404)
    end
  end
end

use Rack::ContentType
run App.new
```

Better... but we can do better by using rack's `URLMap` functionality. We can essentially break up parts of our app into parallel rack apps that only get called if a route matches. The cool thing is that we can set the middleware to run all ALL routes by setting it outside the block, or create our custom stack  for each route.

Here's the updated `config.ru` and HomeController

```ruby
# config.ru
require_relative 'config/environment'
use Rack::ContentType
map '/' do
  run HomeController.new
end

# HomeController
class HomeController
  attr_reader :request
  def call(env)
    @request = Rack::Request.new(env)
    if request.get? && request.path == '/'
      index
    else
      Rack::Response.new("File not found", 404)
    end
  end

  def index
    template = File.read('app/views/home/index.html.erb')
    Rack::Response.new(template)
  end
end
```

This is MUCH better. We had to give the controller it's own `#call` method since it gets called only when this route matches. There is a bit of a smell, and we have a hunch we're heading to the duplication dunes, but we soldier on.

## All the programmers!
Let's make a new controller for our programmers route.

```ruby
# ProgrammersController
class ProgrammersController
  attr_reader :request
  def call(env)
    @request = Rack::Request.new(env)
    if request.get? && request.path == '/programmers'
      index
    else
      Rack::Response.new("File not found", 404)
    end
  end

  def index
    programmers = Programmer.all
    template = File.read('app/views/programmers/index.html.erb')
    result = ERB.new(template).result(binding)
    Rack::Response.new(result)
  end
end
```
And the template

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Potd</title>
</head>
<body>
  <h1>Checkout our programmers!</h1>
  <ul>
    <% programmers.each do |programmer| %>
      <li><a href="/programmers/<%= programmer.id %>"><%= programmer.name %></a></li>
    <% end %>
  </ul>
</body>
</html>

```
Here we have the call method, again, acting as our router, delegating out to our index action or 404ing when it doesn't find anything. Again... this smells, we see some duplication, but we'll hold off. In the immortal words of [Bobby "Crouton" Nadler](http://bobnadler.com/), wait for duplication at least 3 times. By then, you should understand exactly what's being repeated.

Our `config.ru` looks like this now:

```ruby
require_relative 'config/environment'
use Rack::ContentType

map '/programmers' do
  run ProgrammersController.new
end
map '/' do
  run HomeController.new
end
```

### Dynamic Segments - Wrath of the Regex
Ok, so our next requirement is going to make us get... creative. We're going to have to write a router, and write some interesting regular expressions. Let's get started.

The code I wish worked looks something like this

```ruby
# config/routes.rb
Routes = Router.new
Routes.add_route(method: "GET", path: "/programmers", handler: "ProgrammersController#index")
Routes.add_route(method: "GET", path: "/programmers/:id", handler: "ProgrammersController#show")
Routes.add_route(method: "GET", path: "/", handler: "HomeController#index")
```
It would handle any dynamic segment extraction, and call the right controller, modifying params along the way.

This changes our current strategy of using URLMap since the mapping will be made by the router, and not by individual blocks.

### WARNING: LOTS OF CODE
Ok... so don't run. We're friends right? You trust me right? We'll get through this. Just... don't... run...

We'll start with the new `config.ru` file:

```ruby
require_relative 'config/environment'
require_relative 'config/routes'

use Rack::ContentType
run Routes
```

Nice right? It loads all of our routes. `Routes` was made a constant so we could load it in another file and it stay in memory. Basically it's a global. Sue me.

We needed to make a `Router` class to get support this awesome syntax

```ruby
# lib/router.rb
class Router
  attr_reader :request, :routes
  def initialize
    @routes = {}
  end

  def call(env)
    @request = Rack::Request.new(env)
    route = routes[[request.request_method, request.path]]
    if route
      call_action_for(route)
    else
      Rack::Response.new("File not found", 404)
    end
  end

  def add_route(method:, path:, handler:)
    routes[[method, path]] = handler
  end

  private
  def call_action_for(route)
    controller, action = route.split("#")
    controller_class = Kernel.const_get(controller)
    controller_class.new.public_send(action)
  end
end
```

DON'T RUN! Let me explain. You have to think of the router as having 2 big roles. Storing URL, Method combinations, and looking up what to do if it finds a match.

The methods related to that first job are `#initialize` and `#add_route`. Here, we're just giving it the rules. The sexiness is in the `#call` method. Rack will call this method when a request comes in and we'll find a route.

In `#add_route` we're making the key of the hash... an array. You're seeing that right. The value is the string version of the controller#action.

`#call` method asks for the route. Not found? 404, Found? MAGIC.

``` ruby
def call_action_for(route)
  controller, action = route.split("#")
  controller_class = Kernel.const_get(controller)
  controller_class.new.public_send(action)
end
```

Split the string, Find a controller with that name, instantiate and call the action.

### Bad news
This doesn't work for the `/programmers/1` path. All that work... for NOTHING! Let's make Sandi Metz proud and MAKE MOAR OBJECTS! We'll need our "Route" to be smarter than just an array.

### Route object
We need to write our own equality so that we can compare `/programmers/:id` and have it equal `/programmers/1`. We also want to make it so that `/programmers` equals `/programmers` (no dynamic segments here).

Here's our first pass at it

```ruby
class Route
  attr_reader :method, :path
  def initialize(method:, path:)
    @method = method
    @path = path
  end

  def ===(other)
     match_path === other.path && method == other.method
  end

  private

  def match_path
    return path unless has_dynamic_segment?
    Regexp.new(path.gsub(/:\w+/, '(\w+)') + "$")
  end

  def has_dynamic_segment?
    path.include?(":")
  end
end
```
We write a regular expression that replaces any part of the string that starts with a `:` with a `\w+`. Meaning replace `/programmers/:id` into A regular expression containing `/programmers/(\w+)`.

We updated our Router too:

```ruby
class Router
  attr_reader :request, :routes
  def initialize
    @routes = {}
  end

  def call(env)
    @request = Rack::Request.new(env)
    current_route = Route.new(path:request.path, method:request.request_method)
    route, handler = routes.find do |route, handler|
      route === current_route
    end

    if route
      call_action_for(handler)
    else
      Rack::Response.new("File not found", 404)
    end
  end

  def add_route(method:, path:, handler:)
    routes[Route.new(method:method, path:path)] = handler
  end

  private
  def call_action_for(handler)
    controller, action = handler.split("#")
    controller_class = Kernel.const_get(controller)
    controller_class.new.public_send(action)
  end
end
```

This works! We call the right action based on a dynamic segment! We can't stop yet. We're so close. We need to modify the controllers to expect request information so it knows WHICH programmer to show, AND we need to modify the request so it contains the value of the dynamic segment.


Let's do the easy stuff first.
Your home controller should look like this:

```ruby
class HomeController
  attr_reader :request
  def initialize(request)
    @request = request
  end

  def index
    template = File.read('app/views/home/index.html.erb')
    Rack::Response.new(template)
  end
end

```
And your ProgrammersController

```ruby
class ProgrammersController
  attr_reader :request
  def initialize(request)
    @request = request
  end

  def index
    programmers = Programmer.all
    template = File.read('app/views/programmers/index.html.erb')
    result = ERB.new(template).result(binding)
    Rack::Response.new(result)
  end

  def show
    programmer = Programmer.find(request.params["id"])
    template = File.read("app/views/programmers/show.html.erb")
    Rack::Response.new(ERB.new(template).result(binding))
  end
end
```

Then make a template at `app/views/programmers/show.html.erb`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Potd</title>
</head>
<body>
  <h1><%= programmer.name</h1>
  <h2><%= programmer.description %></h2>
</body>
</html>
```

### Router updates
In the router, update the request to include the dynamic segment info.

``` ruby
#...
if route
  route.update_params!(request)
  call_action_for(handler, request)
else
#...
```

Then in the route, add the method.

``` ruby
def update_params!(request)
  return unless has_dynamic_segment?

  keys = path.scan(/:(\w+)/).flatten
  values = request.path.scan(match_path).flatten
  dynamic_segment_params = keys.zip(values).to_h
  request.params.merge!(dynamic_segment_params)
end
```

Do nothing if there's nothing dynamic. Otherwise extract the keys, and values, zip them them merge it into the request's params hash.

### One more thing
Add the Rack::Static Middleware. Any files in /public/css will be served without checking our routes. Great for stylesheets!

```ruby
require_relative 'config/environment'
require_relative 'config/routes'

use Rack::ContentType
use Rack::Static, urls: ['/css'], root: 'public'
run Routes
```

Run this in your terminal. I'm assuming you have `wget` like a respectable developer.

```
mkdir -p public/css
cd public/css
wget https://maxcdn.bootstrapcdn.com/bootstrap/3.3.5/css/bootstrap.min.css
```

Then add this line to the head of all of your html pages

``` html
  <link rel="stylesheet" href="/css/bootstrap.min.css">
```
# That's all!

Next time, we'll handle posting from forms. I hope you found this useful, and enjoyed the journey.
