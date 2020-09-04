---
title: Let's learn about Rack
date: 2015-08-21 14:47 UTC
tags: rack, ruby, basics
---
# Why learn rack?
In Ruby, we're surrounded by magic. From `Sinatra` abstracting the request cycle away, to `rails` making you believe unicorns exist in your terminal, there is no shortage of magical abstraction in Ruby. I've found that uncovering the magic, peeling back the layers of metaprogramming and seeing what's happening is helpful for when your magic turns around and starts attacking you.

We'll be learning a bit of rack, and then move on to building an app with nothing but the most basic of Rack tools.

## What is rack?
The first release of the `rack` gem was released on May 16th, 2007 by Christian Neukirchen to solve a growing problem.

There were webservers, then there were frameworks. Some web servers offered features like concurrency or low level optimizations that were appealing to framework users. Framework authors wanted to support as many webservers as possible so their user base could use whichever they wanted. For every framework, we have to make `x` number of handlers to communicate with these webservers, where `x` is the number of webservers available. I feel like a graphic would be really useful here, so go ahead and imagine one. I'll wait...

![Your sweet mental graphic](/images/insert_image_here.png)

Create a new framework, and you're already behind. In addition to writing the next AMAZING Web framework, you also have to spend time making it work with ALL the existing popular webservers.

### Enter Rack Compliance
Rack Compliance was an agreement. Webservers, follow the standard by formatting your data in a specific way and a Framework will do the same. Imagine a funnel of uniformly formatted data coming in from the request, and leaving from the Framework.

This agreement had a HUGE impact on the ecosystem. Leading to new players to both Frameworks and Webservers to gain adoption, most notably, the `Merb` Framework was able to gain traction and was eventually merged into `Rails 3.0`. Yey standards!

## Overview
Let's get to it. Today we'll be covering the request cycle. From that moment you press `Enter` to the millisecond later when you get your page in the context of `Rack`.

## The request cycle
What does your app receive when your user "sends a request"? Let's say your user is trying to get to the home page. They go to `http://example.com`, and (after some DNS magic) the browser sends you something that looks like this

```
GET / HTTP/1.1
Host: localhost:9292
Connection: keep-alive
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/44.0.2403.155 Safari/537.36
Accept-Encoding: gzip, deflate, sdch
Accept-Language: en-US,en;q=0.8,es-419;q=0.6,es;q=0.4
```

If you want to try this your self, run [this](https://gist.github.com/octosteve/17487d85c5493fb4404e) script and visit `http://localhost:9292` in an incognito browser.

Ok, so you get this and what are are you supposed to do with it? If you're working with a [Rack](https://www.digitalocean.com/community/tutorials/a-comparison-of-rack-web-servers-for-ruby-web-applications) Compliant web server, it's going to take this information and parse it into something your app can more easily work with: A Hash.

To see what this looks like to your `Rack` app, let's make a simple one.

All we have to do to make a `Rack` app is create an object that responds to the `call` method and takes one argument. We'll do that by making an `App` class with a `call` instance method. We'll make it respond with an array of 3 values: The status code, any headers, and the body in an array. Stick this in a file named `config.ru`

```ruby
# config.ru
class App
  def call(env)
    p env
    [200, {"Content-Type" => "text/html"}, ["Check your console"]]
  end
end

run App.new
```

Run `rackup` and visit `http://localhost:9292`. You should see something like this:

```ruby
{
                 "GATEWAY_INTERFACE" => "CGI/1.1",
                         "PATH_INFO" => "/",
                      "QUERY_STRING" => "",
                       "REMOTE_ADDR" => "127.0.0.1",
                       "REMOTE_HOST" => "localhost",
                    "REQUEST_METHOD" => "GET",
                       "REQUEST_URI" => "http://localhost:9292/",
                       "SCRIPT_NAME" => "",
                       "SERVER_NAME" => "localhost",
                       "SERVER_PORT" => "9292",
                   "SERVER_PROTOCOL" => "HTTP/1.1",
                   "SERVER_SOFTWARE" => "WEBrick/1.3.1 (Ruby/2.2.2/2015-04-13)",
                         "HTTP_HOST" => "localhost:9292",
                   "HTTP_CONNECTION" => "keep-alive",
                "HTTP_CACHE_CONTROL" => "max-age=0",
                       "HTTP_ACCEPT" => "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8",
    "HTTP_UPGRADE_INSECURE_REQUESTS" => "1",
                   "HTTP_USER_AGENT" => "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/44.0.2403.155 Safari/537.36",
              "HTTP_ACCEPT_ENCODING" => "gzip, deflate, sdch",
              "HTTP_ACCEPT_LANGUAGE" => "en-US,en;q=0.8,es-419;q=0.6,es;q=0.4",
                      "rack.version" => [
        [0] 1,
        [1] 3
    ],
                        "rack.input" => #<Rack::Lint::InputWrapper:0x007fd48c8d05f0 @input=#<StringIO:0x007fd48c8d84a8>>,
                       "rack.errors" => #<Rack::Lint::ErrorWrapper:0x007fd48c8d05c8 @error=#<IO:<STDERR>>>,
                  "rack.multithread" => true,
                 "rack.multiprocess" => false,
                     "rack.run_once" => false,
                   "rack.url_scheme" => "http",
                      "rack.hijack?" => true,
                       "rack.hijack" => #<Proc:0x007fd48c8d0ac8@/Users/octosteve/.gem/ruby/2.2.2/gems/rack-1.6.4/lib/rack/lint.rb:525>,
                    "rack.hijack_io" => nil,
                      "HTTP_VERSION" => "HTTP/1.1",
                      "REQUEST_PATH" => "/",
                  "rack.tempfiles" => []
}

```

This looks a lot like what the browser sent us! What method did we use? Check `env["REQUEST_METHOD"]`. Want to know where they were going? Check `env["PATH_INFO"]`.

Simple. We get a Hash, and return an array... but EW, we get a HASH and return an ARRAY?!

We know a thing or two about [Primitive Obsession](http://c2.com/cgi/wiki?PrimitiveObsession), so let's wrap these in objects. Lucky for us, `Rack` comes with a ways to wrap them nicely. `Rack::Request`, and `Rack::Response`.

Let's upgrade our code to look like this:

``` ruby
class App
  def call(env)
    request = Rack::Request.new(env)
    Rack::Response.new("You made a #{request.request_method} request to #{request.path_info}", 200, {"Content-Type" => "text/html"})
  end
end

run App.new
```

Visit `http://localhost:9292` and you'll see `You made a GET request to /`. Visit `http://localhost:9292/banana` and you'll see `You made a GET request to /banana`.

We can reduce a bit of code by taking advantage of some Rack defaults, and the `Rack::ContentType` middleware. Let's modify our code to look like this

``` ruby
class App
  def call(env)
    request = Rack::Request.new(env)
    Rack::Response.new("You made a #{request.request_method} request to #{request.path_info}")
  end
end

use Rack::ContentType
run App.new
```

`Rack::ContentType` sets all outgoing responses to be 'text/html'. If you need to override this, say to have it always return `application/json`, try `use Rack::ContentType, 'application/json'` instead.

`Rack::Response` sets its second constructor argument to `200` by default, so we can leave it out. So clean!

### Dealing with post requests and request bodies

Load up the `rest-client` gem in the console and try this.

``` ruby
# make sure you've run gem install rest-client before running this
require 'rest-client'
RestClient.post("http://localhost:9292/banana", {potassium: "High"}).body
# => "You made a POST request to /banana"
```
It works... but how do we work with those parameters? Come on Rack, don't be stingy, let me see those sweet sweet params I sent you bruh. Let's update our code to read the request's body. According to the [RFC](http://www.w3.org/Protocols/rfc2616/rfc2616-sec4.html) (try reading these if you're having trouble sleeping) the body of a request is "used to carry the entity-body associated with the request or response". Basically, the goods we're after. We can read them with rack by using the `body` method on a request object. Let's update the code to output we sent in our post request.

``` ruby
class App
  def call(env)
    request = Rack::Request.new(env)
    Rack::Response.new("You made a #{request.request_method} request to #{request.path_info} here's the body #{request.body.read}")
  end
end

use Rack::ContentType
run App.new
```
And then back in `irb`

``` ruby
RestClient.post("http://localhost:9292/banana", {potassium: "High"}).body
# => "You made a POST request to /banana here's the body potassium=High"
```

Sweet11!!1! Except... I gave you a hash, and you gave me a mashed up string. WTF rack. Easy fix, though. We can use `Rack::Utils.parse_nested_query` to get it into an easier form for us to work with.

``` ruby
class App
  def call(env)
    request = Rack::Request.new(env)
    body = Rack::Utils.parse_nested_query(request.body.read)
    Rack::Response.new("You made a #{request.request_method} request to #{request.path_info} here's the body #{body}")
  end
end

use Rack::ContentType
run App.new
```
And...

```ruby
RestClient.post("http://localhost:9292/banana", {potassium: "High"}).body
# => "You made a POST request to /banana here's the body {\"potassium\"=>\"High\"}"
```

Now we're cooking with gas!

# That's it for the basics
Next time, we'll use these tools to build an app. We'll implement multiple routes, create controllers, and use some other really powerful Rack middleware and features.


Happy Clacking!
