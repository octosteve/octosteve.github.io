---
title: LTrees in Phoenix
date: 2016-05-22 16:38 UTC
tags: ["Elixir", "Phoenix", "Ecto", "LTrees"]
---

# Article Tracker

Say you want to build a Phoenix app for tracking Articles. Your one killer feature
is that you can store articles in a hierarchy instead of a bunch loose tags like other sites. We'll be focusing on the model layer here, but we'll see how we'd construct the query for our tree

You want to be able to click an article with `Technology > Futurism > AI > Global Domination`, then click on `AI` in the header, and be taken to a list of articles with `Technology > Futurism > AI` which might include on other `AI` things, not related to them trying to kill us all.


## Data Modeling
Nothing crazy here. We'll have an Article Model with a few properties.

| id | title | url | categories |
| ------------- | ------------- | ------------- | ------------- |
| 1 | How AI took over the world      |  http://bit.ly/1EKZyGW | technology.futurism.ai.global_domination |
| 2 | How AI are good for the elderly |  http://bit.ly/27ONtlY | technology.futurism.ai.helping |


To get nested categories to work, we're going to use a data structure offered in Postgres, the `ltree`. This blog post will focus on getting this working in Phoenix. For more information, I invite you to [Read The Fine Manual](http://www.postgresql.org/docs/9.3/static/ltree.html). One thing to consider from the Manual is what makes a valid ltree value:


>A label is a sequence of alphanumeric characters and underscores. Labels must be less than 256 bytes long.


## Error Driven Development
Let's build this and fix errors as they come up.

First, generate a new Phoenix app

```
mix phoenix.new article_tracker_hd
cd article_tracker_hd
mix ecto.create
```

Then generate a quick scaffold that gets us most of the way there.
`mix phoenix.gen.html Article articles title url categories`

Add your resource to your router

``` elixir
#...
scope "/", ArticleTrackerHd do                                                   
  pipe_through :browser # Use the default browser stack                          

  get "/", PageController, :index                                                
  resources "/articles", ArticleController # <- New line
end
#...
```

Then open up the migration in `priv/repo/migrations/{some_time_stamp}_create_article.exs`. This is where the fun starts.

Update your migration to look like this:

``` elixir
defmodule ArticleTrackerHd.Repo.Migrations.CreateArticle do
  use Ecto.Migration

  def change do
    create table(:articles) do
      add :title, :string
      add :url, :string
      add :categories, :ltree # <- Changed to :ltree

      timestamps
    end
    create index(:articles, [:categories], using: "GIST") # <- Add indexing for fast lookups

  end
end
```
Then run the migrations with `mix ecto.migrate`

You'll be greeted with a wonderful error "** (Postgrex.Error) ERROR (undefined_object): type "ltree" does not exist". Time to go home.

First we have to open our database, and create the ltree extension. We'll have to do this for every database we use including test and production. We'll do it in article\_tracker\_hd_dev for now.

```
psql article_tracker_hd_dev
article_tracker_hd_dev=# CREATE EXTENSION ltree;
article_tracker_hd_dev=# \q;
```

And try again `mix ecto.migrate`. It WORKED! Let's open up the console and try to insert an article.

```
iex -S mix
iex(1)> alias ArticleTrackerHd.{Article, Repo}
iex(2)> article = %Article{title: "How AI took over the world", url: "http://www.computerworld.com/article/2922442/robotics/stephen-hawking-fears-robots-could-take-over-in-100-years.html", categories: "technology.futurism.ai.global_domination"}
iex(3)> Repo.insert!(article)
** (ArgumentError) no extension found for oid `342415`
```

MOAR Errors! This just means Postgrex doesn't know how to map ltrees. We can fix this by adding a new Postgex type for `ltree`. Make a new file in `lib/postgrex/types/` called `ltree.ex`, and insert this code:

``` elixir
defmodule ArticleTrackerHd.Postgrex.Types.Ltree do
  alias Postgrex.TypeInfo

  @behaviour Postgrex.Extension

  def init(_parameters, _opts), do: {}

  def matching(_), do: [type: "ltree"]

  def format(_), do: :text

  def encode(%TypeInfo{type: "ltree"}, value, _state, _opts), do: value

  def decode(%TypeInfo{type: "ltree"}, value, _state, _opts), do: value
end
```

A lot of this is ceremony, but the things to note are the format, encode and decode functions.
Format has 2 possible return values "text", and "binary". We're just storing the categories as text so that will do. Encode and decode are your data gateways. Right now, I'm ok just presenting a set of categories as a string of contiguous characters with dots in between.

We'll also have to tell Postgrex to load our extension. Open up `config/dev.exs` and add this to where you configure your database:

``` elixir
config :article_tracker_hd, ArticleTrackerHd.Repo,
  adapter: Ecto.Adapters.Postgres,
  username: "postgres",
  password: "postgres",
  database: "article_tracker_hd_dev",
  hostname: "localhost",
  pool_size: 10,
  extensions: [{ArticleTrackerHd.Postgrex.Types.Ltree, []}] # <- New line
```

You'll have to do this for every environment you run this code in.

Try to add the article again:

```
iex -S mix
iex(1)> alias ArticleTrackerHd.{Article, Repo}
iex(2)> article = %Article{title: "How AI took over the world", url: "http://www.computerworld.com/article/2922442/robotics/stephen-hawking-fears-robots-could-take-over-in-100-years.html", categories: "technology.futurism.ai.global_domination"}
iex(3)> Repo.insert!(article)

```

WIN! On to the cool part.

## Querying categories
Add these additional stories in `iex`

```
iex -S mix
iex(1)> alias ArticleTrackerHd.{Article, Repo}
iex(2)> article = %Article{title: "How AI are good for the elderly ", url: "http://thevitalityinstitute.org/innovations-artificial-intelligence-elderly/", categories: "technology.futurism.ai.helping"}
iex(3)> Repo.insert!(article)
iex(4)> article = %Article{title: "Google’s AI Wins Fifth And Final Game Against Go Genius Lee Sedol", url: "http://www.wired.com/2016/03/googles-ai-wins-fifth-final-game-go-genius-lee-sedol/", categories: "technology.futurism.ai.winning"}
iex(5)> Repo.insert!(article)
```

Then let's work on some queries. We'll be using the [`fragment`](https://hexdocs.pm/ecto/Ecto.Query.API.html#fragment/1) function to get our `ltree` search working. Say we wanted to find all of the articles starting with technology:

```
iex(6)> import Ecto.Query
iex(7)> query = from a in Article, where: fragment("categories <@ ?", "technology")
iex(8)> Repo.all(query)
```

This should return all of the articles so far. The `<@` asks for all categories that have start with `technology`, if you try it with `ai` you'll get nothing.

Let's be more specific and get the ones that are `technology.futurism.ai.winning`:

```
iex(9)> query = from a in Article, where: fragment("categories <@ ?", "technology.futurism.ai.winning")
iex(10)> Repo.all(query)
```

How about all of the ai stories regardless of where they show up on the category tree? Let's add another article, then search:

```
iex(11)> article = %Article{title: "How AI are beating you at rock paper scissors", url: "http://www.nytimes.com/interactive/science/rock-paper-scissors.html?_r=0", categories: "technology.gaming.ai.winning"}
iex(12)> Repo.insert!(article)
iex(13)> query = from a in Article, where: fragment("categories ~ ?", "*.winning.*")
iex(14)> Repo.all(query)      
```

We used the `~` to indicate we were going use path matching. Tons of options here, be sure to check out the ltree page on the Postgres guide.

# Conclusion
LTrees are my favorite thing right now. A few things I'd like to see is smarter conversion of the values to and from the db and validating the ltree is compliant in a changeset. Bad characters drive it bananas.

We didn't use an [Ecto.Type](https://hexdocs.pm/ecto/Ecto.Type.html) annotation, so we're using a "String" as far as Ecto is concerned. If I had any addition conversion so it played nice with my domain, I'd do it there. For instance, making it so users can give you something like "Technology > Computers > Programming > Elixir" and have it converted over to "technology.computers.programming.elixir" would be in an Ecto.Type.

Found this interesting? Be sure to share it!

<a href="https://twitter.com/share" class="twitter-share-button" data-text="Using LTrees in Phoenix. Hierarchical data in Postgres" data-via="_StevenNunez" data-size="large">Tweet</a>

<script>!function(d,s,id){var js,fjs=d.getElementsByTagName(s)[0],p=/^http:/.test(d.location)?'http':'https';if(!d.getElementById(id)){js=d.createElement(s);js.id=id;js.src=p+'://platform.twitter.com/widgets.js';fjs.parentNode.insertBefore(js,fjs);}}(document, 'script', 'twitter-wjs');</script>
