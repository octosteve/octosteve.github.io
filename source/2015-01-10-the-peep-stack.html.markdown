---
title: The PEEP Stack
date: 2015-01-10 17:05 UTC
tags: Postgres, Ember, Elixir, Phoenix Framework
---
# PEEP Stack

Today I want to talk about building apps with the PEEP Stack. Postgres, Ember, Elixir, and the Phoenix Framework.

## Meet the team Postgres
[Postgres](http://www.postgresql.org/) is an extremely powerful RDMBS that supports clusturing, has full text search built in, and support for spacial data with PostGIS. Postgres is a beast. It's my database of choice.

## Ember
Let's face it. The Javascript landscape is insanely volitile. Libraries die, or face huge rewrites from [version to version](eisenbergeffect.bluespire.com/all-about-angular-2-0). It's hard to keep up.

> We believe it's possible to achieve stability without stagnation.
>
> [Yehuda Katz](https://twitter.com/wycats)

Ember gets it. I've taken apps written with a pre v1 version, swapped the version to the latests release, followed the deprecation notices in the console, and had it using the latest and greatest code. I'm impressed.

In addition, Ember is deeply conventions based, meaning there's one true way to do something. I find this really appealing.

## Elixir

I'm really excited about Elixir. It's a beautiful language that takes a lot of the best features of today's modern programming languages. Check out [this talk](http://www.confreaks.com/videos/5078-RubyConf2014-rubyists-have-a-sip-of-elixir) from RubyConf to get an idea of its amazingness.

Another benefit of working with Elixir is you get to take advantage of Erlang's ecosystem. Erlang was built for distributed, concurrent, and scalable programs. One of the most recent Erlang success stories was [What's app](https://www.whatsapp.com). You can watch a great talk on Erlang's scalability [here](https://www.youtube.com/watch?v=c12cYAUTXXs).

### Performance YAGNI
Before I go further, I'm sure some of you are thinking "My app probably won't HAVE to support a billion users. Why bother writing it in a platform that's built for those apps?". Good point anonymous internet Joe. I'd agree with you if Elixir wasn't such a joy to write. If you CAN have this amazing scalability option, at no real cost to you, why wouldn't you take it?

## The Phoenix Framework
[The Phoenix Framework](http://www.phoenixframework.org/) is game changing. Modeling itself after [Ruby on Rails](http://rubyonrails.org/) means moving over as a Rails dev is insanely simple. Having [José Valim](https://twitter.com/josevalim) involved is bringing in some Rails conventions which I love.


## Build time
Let's set up a basic app with Phoenix and Ember. For Phoenix, we'll be using the built in app generator, and for Ember, we'll be using [Ember-Cli](ember-cli.com). We'll be building the __Hello World__ of the web, a __chat room__. It was a blog, but the world loves async stuff now.

### New Phoenix App

I'm assuming you have Elixir installed. If not, go [here](http://elixir-lang.org/getting_started/1.html).

Installing Phoenix is easy. Run:

``` bash
git clone https://github.com/phoenixframework/phoenix.git
cd phoenix
git checkout v0.7.2
mix do deps.get, compile
```
Checkout the [guide](http://www.phoenixframework.org/) for more details on what this does.

Now to create a new app. From inside this directory run:


``` bash
mix phoenix.new peep_chat ../peep_chat
cd ../peep_chat
mix do deps.get, compile
```

We'll modify a few things:

* delete the `page` directory in `web/templates/`. We won't be rendering views with Phoenix, just an api.
* delete `web/views/page_view.ex`
* open `web/router.ex` and change references to PageController to HomeController.
* Rename `web/controllers/page_controller.ex` to `web/controllers/home_controller.ex`
* Change the controller name in `home_controller.ex` to `PeepChat.HomeController`

We'll leave this here for now. After we have Ember set up, we'll have this action load Ember's generated `index.html`.

## Ember-Cli

Ember-Cli will be managing our assets. This allows us to work with stuff like Sass and Coffeescript with ease. I love this separation since we leave Ember-Cli to do what it's good at: wrangling the JS chaos.

We'll create a new Ember app inside our peep_chat app's web directory named client.

``` bash
cd web
ember new client
```

The next step we'll need is to change the configurations so Ember's build step outputs to Phoenix's `priv/static` directory.

Add this to `peep_chat/web/client/.ember-cli` under the `disableAnalytics` setting.:

``` json
"output-path": "../../priv/static"
```

By running `ember build` you should see a bunch of Ember files dumped in Phoenix's `priv/static` directory.

### Serving up Ember
One last step to get the Ember generated page served. Open up `web/controllers/home_controller.ex` and replace your index action with the following:

``` elixir
def index(conn, _params) do
  {:ok, home_page} = File.read 'priv/static/index.html'
  html conn, home_page
end
```

This reads the Ember generated index file and returns it when someone asks for the root path.

Fire up the Phoenix server with `mix phoenix.start`, and visit `http://localhost:4000`. There you should see the Ember Landing page.

### Push State and routing
Ember's going to use HTML pushState by default meaning you won't get urls with #'s. For this to work, we're going to change a line in our route file to act as a catch all, this way if the user asks for / or /rooms/1, our app will always serve Ember's `index.html` file. We'll add to this later when we build our API out.

Open `web/router.ex` and edit this:

``` elixir
scope "/", PeepChat do
  pipe_through :browser # Use the default browser stack

  get "/", HomeController, :index
end
```

to

``` elixir
scope "/", PeepChat do
  pipe_through :browser # Use the default browser stack

  get "/*path", HomeController, :index
end
```

This wild force all requests to respond with Ember's `index.html`
