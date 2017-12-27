---
title: Guard this with your life... Or authenticating APIs with Guardian
date: 2016-05-23 21:58 UTC
tags: elixir, phoenix
---
# Authentication in Phoenix
Conceptually, authentication isn't hard. You collect a username and password, check it against your database and if it matches, WIN!

![getinsertpic.com](http://media2.giphy.com/media/3BlN2bxcw0l5S/200.gif)

... then you get into the business of persisting that information across requests, not to mention all of the security concerns of storing a password safely. In this post we'll be covering how to authenticate users, provide them with a token, and use that token on subsequent requests to identify our user. There are a TON of moving parts, so strap in.

## Overview

We'll be covering:

* Modeling an Account
* Using changesets
* Hashing passwords with [`comeonin`](https://github.com/elixircnx/comeonin)
* Managing "impure" transactions in Service Modules
* Using Guardian to manage token sharing
* Identifying the logged in user

## The App
We'll be expanding on our Article Tracker located [here](https://github.com/StevenNunez/article_tracker_hd/tree/authentication-blog-start).

## Accounts

Accounts will have an email and a password_digest. The digest will hold our hashed password... eventually.

Run `mix phoenix.gen.model Account accounts email password_digest` then `mix ecto.migrate`.

Open the new model, and update the schema to have a virtual attribute:

```elixir
schema "accounts" do
  field :email, :string
  field :password_digest, :string
  field :password, :string, virtual: true # <- New line

  timestamps
end
```

Virtual attributes are a great way of taking a value from an interaction, but not persisting. In our case, we'll be using it to hold the value a users gives us before we hash and persist it.

# Changesets
Changesets are a way of describing the process a set of proposed changes go through in preparation to being persisted. You define your rules, then try to persist. If everything is good to go, say passing all of your validation rules, it can be inserted into your database.

Storing passwords in plain text is a bad, NAY *TERRIBLE* IDEA!! We want to allow a user to send us a password, that we'll hash then store.

Creating a changeset is easy. In the model, we call `cast` and pass in the model, a map with proposed values, and the fields we require, and permit.

This is the built in one from our project. Be sure to change the `password_digest` in our `@required_fields` to `password`. The user will never pass us a `password_digest`

``` elixir
@required_fields ~w(email password) # <- Changed from password_digest to password
@optional_fields ~w()

def changeset(model, params \\ :empty) do
  model
  |> cast(params, @required_fields, @optional_fields)
end
```
This returns a changeset back. We can tag along another function, as long as it returns a changeset as well. Let's do some work to obfuscate the user's password.

```elixir
def changeset(model, params \\ :empty) do
  model
  |> cast(params, @required_fields, @optional_fields)
  |> put_pass_hash
end

defp put_pass_hash(changeset) do
  password = changeset.changes.password
  put_change(changeset, :password_digest, String.reverse(password))
end
```

Open up `iex -S mix` and try this out:

```
iex> alias ArticleTrackerHd.Account
iex> changeset = Account.changeset(%Account{}, %{email: "steven@example.com", password: "beef101"})
iex> changeset.changes.password_digest # => "101feeb"
```

\#security \#hashtag. I think we need to be a bit more deliberate in our approach.

# Come on in!
Maybe not what you want to hear in a security library, but, hey... We'll be adding this library to handle our password hashing.

Checkout their [github](https://github.com/elixircnx/comeonin) and add the latest version to your `mix.exs` file.

``` elixir
defp deps do
  [{:phoenix, "~> 1.1.4"},
   {:postgrex, ">= 0.0.0"},
   {:phoenix_ecto, "~> 2.0"},
   {:phoenix_html, "~> 2.4"},
   {:phoenix_live_reload, "~> 1.0", only: :dev},
   {:gettext, "~> 0.9"},
   {:comeonin, "~> 2.4"}, # <- New line
   {:cowboy, "~> 1.0"}]
end


```

Then start the application:

``` elixir
def application do
  [mod: {ArticleTrackerHd, []},
   applications: [
     :phoenix,
     :phoenix_html,
     :cowboy,
     :logger,
     :gettext,
     :phoenix_ecto,
     :postgrex,
     :comeonin # <- New Line
    ]
  ]
end
```

`mix deps.get` and we're good to go.

### Updating our changeset
Let's update to our super secure hashing algorithm.

```elixir
defp put_pass_hash(changeset) do
  password = changeset.changes.password
  put_change(changeset, :password_digest, Comeonin.Bcrypt.hashpwsalt(password))
end
```

Open up `iex -S mix` and try again:

```
iex> alias ArticleTrackerHd.{Account, Repo}
iex> changeset = Account.changeset(%Account{}, %{email: "steven@example.com", password: "beef101"})
iex> changeset.changes.password_digest # => "Something crazy"
iex> Repo.insert!(changeset)
```

WIN! We're doing right by our users. Note that we saved user with an email of `steven@example.com` and a password of `beef101`.

### Authenticating users
If you're coming from rails, you might be tempted to slap on an `authenticate` function and have it find the account, check the password and all that jazz. Well HOLD YOUR HORSES!
![getinsertpic.com](https://media3.giphy.com/media/ZvRz1cnhjyecw/200.gif)

In Phoenix, we create "pure" function in our models and views, and "impure" functions... elsewhere. What is a "pure" function? Think of a function that takes in 2 values

``` elixir
fn (a, b) -> a + b end
```

If you call this function with 1 and 2, the result will always be the same. Now imagine calling this code:

```elixir
Repo.all(Account)
```

This code might return 1 User, or 300. We're touching the outside world. Fancy terms for simple things.

Back to authentication. We need a place to handle our impure operations. I'm coining a new term, We need a *Service Module*

![getinsertpic.com](http://media0.giphy.com/media/GsYaDdbX7k3Kw/200.gif)

This module will hold our impure auth related stuff. Make a new file in `web/services` called `auth.ex`.

``` elixir
defmodule ArticleTrackerHd.Auth do
  alias ArticleTrackerHd.{Repo, Account}
  import Comeonin.Bcrypt, only: [checkpw: 2, dummy_checkpw: 0]

  def verify(email, password) do
    account =  Repo.one(Account.find_by_email(email)) # <- Need to define find_by_email
    cond do
      account && checkpw(password, account.password_digest) -> # <- Valid account and password!
        {:ok, account}
      account -> # <- Found account but password didn't match
        {:error, :bad_password}
      true ->
        dummy_checkpw # <- Fool hackers
        {:error, :not_found}
    end
  end
end
```

So IMPURE! We have one nugget of purity and that's in the `find_by_email` function. That will return the query we'll pass to `Repo.one`. Here it is:

```elixir
def find_by_email(email) do
  from a in __MODULE__,
  where: a.email == ^email
end
```

Try it out. `iex -S mix`.

```
iex> alias ArticleTrackerHd.{Account, Auth}
iex> Auth.verify("steven@example.com", "beef101")
{:ok,
 %ArticleTrackerHd.Account{__meta__: #Ecto.Schema.Metadata<:loaded>,
  email: "steven@example.com", id: 1,
  inserted_at: #Ecto.DateTime<2016-05-24T02:29:27Z>, password: nil,
  password_digest: "$2b$12$vEJ80G.uEpJapbXgsGHBN.aZSgDD3yyWiQm.gimOSZimApDL.pTOm",
  updated_at: #Ecto.DateTime<2016-05-24T02:29:27Z>}}
iex> Auth.verify("steven@example.com", "wrongpw")
{:error, :bad_password}
iex> Auth.verify("nope@example.com", "wrongpw")  
{:error, :not_found}
```

GLORIOUS!!

# Guardian
Guardian is a way of generating [JWT](https://jwt.io) [tokens](https://tools.ietf.org/html/rfc7519). Tokens can be read from the session, or from the headers. We're designing this to work with an API, so we'll check the headers.

Guardian doesn't manage finding accounts, that's your job, hence all the blah blah from before. Once that's all set up, we' can leverage Guardian to manage keeping our user requests and our app in sync.

Let's set it up. You'll need a key for the next part. Run `mix phoenix.gen.secret` and hold on to the value.

Open up `mix.exs`:

```  elixir
defp deps do
  [{:phoenix, "~> 1.1.4"},
   {:postgrex, ">= 0.0.0"},
   {:phoenix_ecto, "~> 2.0"},
   {:phoenix_html, "~> 2.4"},
   {:phoenix_live_reload, "~> 1.0", only: :dev},
   {:gettext, "~> 0.9"},
   {:comeonin, "~> 2.4"},
   {:guardian, "~> 0.10.0"}, # <- New line
   {:cowboy, "~> 1.0"}]
end
```
Run `mix deps.get` to get setup.

We'll need to create a configuration for Guardian in our `config.exs`.

``` elixir
config :guardian, Guardian,
  issuer: "ArticleTrackerHd",
  ttl: { 30, :days },
  secret_key: "46l5yfRFGorxAArf64nGzHlfvSDAOUEV7m6c3/lf4LzZcBfUClDsSfNETrosmsdO", # <- Your Key
  serializer: ArticleTrackerHd.GuardianSerializer # <- We'll create this next
```

OK, so a bunch of stuff is about to happen... Brace yourself.

![getinsertpic.com](http://media3.giphy.com/media/psK7tSNSAE8Te/200.gif)

We're going to add 2 plugs. One to teach Guardian how to find our token, and another that lets us convert the token to an account.

Open up your router and add:

```elixir
pipeline :api do
  plug :accepts, ["json"]
  plug Guardian.Plug.VerifyHeader, realm: "Bearer" # <- New line (1)
  plug Guardian.Plug.LoadResource                  # <- New line (2)
end
```

The first line tells Guardian to look in the header for `Authorization: Bearer some_crazy_token`. If you didn't provide a realm, it would expect `Authorization: some_crazy_token`.

The second line works in tandem with the _not yet written_ `ArticleTrackerHd.GuardianSerializer` to convert a token value into a record.

Let's go write that serializer. Create a file in `web/serializers` named `guardian_serializer.ex`

``` elixir
defmodule ArticleTrackerHd.GuardianSerializer do
  @behaviour Guardian.Serializer

  alias ArticleTrackerHd.Repo
  alias ArticleTrackerHd.Account

  def for_token(account = %Account{}), do: { :ok, "Account:#{account.id}" }
  def for_token(_), do: { :error, "Unknown resource type" }

  def from_token("Account:" <> id), do: { :ok, Repo.get(Account, id) }
  def from_token(_), do: { :error, "Unknown resource type" }
end
```

Nothing too crazy here. If you find a claim in the token like `Account:1`, find account with an id of 1.

That's it for Guardian Setup!

## Logging in
![getinsertpic.com](http://media2.giphy.com/media/xTiTngXiWGXLmIZVYY/200.gif)

Let's tie this all together. We'll create a `SessionController` with a `create` action. It will check if the username and password are valid and if it is, we'll sign them in.

Modify your router:

``` elixir
scope "/api", ArticleTrackerHd do
  pipe_through :api
  resources "/articles", ArticleController, except: [:new, :edit]
  post "/log-in", SessionController, :create # <- New line
end
```

Then make a file in `web/controllers/` named `session_controller.ex`.

``` elixir
defmodule ArticleTrackerHd.SessionController do
  use ArticleTrackerHd.Web, :controller
  alias ArticleTrackerHd.Auth

  def create(conn, %{"account" => %{"email" => email, "password" => password}}) do
    case Auth.verify(email, password) do # <- 1
      {:ok, account} ->
         conn
         |> sign_in(account)
         |> render("login.json")
      {:error, _} ->
        conn
        |> put_status(:unprocessable_entity)
    end
  end

  defp sign_in(conn, account) do # <- 2
     conn
     |> Guardian.Plug.api_sign_in(account)
     |> add_jwt
  end

  defp add_jwt(conn) do # <- 3
    jwt = Guardian.Plug.current_token(conn)
    assign(conn, :jwt, jwt)
  end
end

```

Create a file named `session_view.ex` in `web/views/`:

```elixir
defmodule ArticleTrackerHd.SessionView do
  use ArticleTrackerHd.Web, :view

  def render("login.json", %{jwt: jwt}) do
    %{ jwt: jwt }
  end
end
```


The important parts are are numbered.

1. We're using our Auth Module!
2. We're using the Guardian.Plug.api\_sign\_in function. This will call our GuardianSerializer and add the account to the token.
3. This adds the generated jwt to the `conn` so we can use it in our `login.json`.


Let's test it out!


``` bash
curl --data "account[email]=steven@example.com&account[password]=beef101" http://localhost:4000/api/log-in
# {"jwt":"eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJBY2NvdW50OjEiLCJleHAiOjE0NjY2NTQxMzUsImlhdCI6MTQ2NDA2MjEzNSwiaXNzIjoiQXJ0aWNsZVRyYWNrZXJIZCIsImp0aSI6IjhlOTE2ZjQ2LWM5MmEtNDBhMi1iYzE0LTY3Y2VjMmEyYzI1OSIsInBlbSI6e30sInN1YiI6IkFjY291bnQ6MSIsInR5cCI6InRva2VuIn0.OXK_Hu6uJRo3QClBdY05_xFuUdXK2NS3oTCRNZFd6yhyyl-ub1bjBZQV-flM3J6i7WIh1QUvTV4L7I6x72VADA"}
```

WOOOOOOOOT! Now we just need to make a request with that token, and we should be good to go.

## User Autorization
So we're logged in. Now we want to restrict creating new articles to logged in users, but we want all users to be able to see them.

Open up the `ArticleTrackerHd.ArticleController`.

```elixir
defmodule ArticleTrackerHd.ArticleController do
  use ArticleTrackerHd.Web, :controller

  alias ArticleTrackerHd.Article

  plug Guardian.Plug.EnsureAuthenticated, [handler: ArticleTrackerHd.GuardianErrorHandler]  when action in [:create, :update, :delete] # <- New line
  plug :scrub_params, "article" when action in [:create, :update]
#...
end
```

This line makes it so only authenticated users can create articles. We'll have to create a new controller for handling guardian errors. If it fails authentication, it will call the `unauthenticated` function in the module you give it.

Put this in `web/controllers/guardian_error_handler`:

```elixir

defmodule ArticleTrackerHd.GuardianErrorHandler do
  use ArticleTrackerHd.Web, :controller
  alias ArticleTrackerHd.GuardianErrorView
  def unauthenticated(conn, _params) do
      conn
      |> put_status(:forbidden)
      |> render(GuardianErrorView, :forbidden)
    end
end
```

If it fails we want to render a view that returns a JSON message of forbidden. Make a new file in `web/views` named `guardian_error_view.ex`:

```elixir
defmodule ArticleTrackerHd.GuardianErrorView do
  use ArticleTrackerHd.Web, :view
  def render("forbidden.json", _assigns) do
    %{message: "Forbidden"}
  end
end
```

Let's try this out, then get a beer.

``` bash
curl --data "account[email]=steven@example.com&account[password]=beef101" http://localhost:4000/api/log-in
# {"jwt":"eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJBY2NvdW50OjEiLCJleHAiOjE0NjY2NTQ2NzUsImlhdCI6MTQ2NDA2MjY3NSwiaXNzIjoiQXJ0aWNsZVRyYWNrZXJIZCIsImp0aSI6IjBjZDNiYWZhLTY4MjYtNGJlNi1hYWM3LTMzZWJkNTAwMzM5NyIsInBlbSI6e30sInN1YiI6IkFjY291bnQ6MSIsInR5cCI6InRva2VuIn0.S3jQkmtia-mlAv0O0u1QZxWr3MFH5IBnG1C_u0Uyjq6-DOA5is3l8tKiI0M83Vfw5ADUi55uXfoRqYRY7EQ-DQ"}
curl http://localhost:4000/api/articles
# all the articles
curl --data "article[url]=http://example.com&article[title]=A+Great+one&article[categories]=foo.bar.baz" http://localhost:4000/api/articles
# Forbidden
curl --data "article[url]=http://example.com&article[title]=A+Great+one&article[categories]=foo.bar.baz" http://localhost:4000/api/articles --header "Authorization: Bearer eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJBY2NvdW50OjEiLCJleHAiOjE0NjY2NTQ2NzUsImlhdCI6MTQ2NDA2MjY3NSwiaXNzIjoiQXJ0aWNsZVRyYWNrZXJIZCIsImp0aSI6IjBjZDNiYWZhLTY4MjYtNGJlNi1hYWM3LTMzZWJkNTAwMzM5NyIsInBlbSI6e30sInN1YiI6IkFjY291bnQ6MSIsInR5cCI6InRva2VuIn0.S3jQkmtia-mlAv0O0u1QZxWr3MFH5IBnG1C_u0Uyjq6-DOA5is3l8tKiI0M83Vfw5ADUi55uXfoRqYRY7EQ-DQ"
curl http://localhost:4000/api/articles
```
VICTORY!

## Quick associations
Let's add a reference to users on articles. Then we'll create them and attribute them.

Run `mix ecto.gen.migration add_account_reference_to_article`

Then modify it to say:

```elixir
defmodule ArticleTrackerHd.Repo.Migrations.AddAccountReferenceToArticle do
  use Ecto.Migration

  def change do
    alter table(:articles) do
      add :account_id, references(:accounts)
    end
  end
end
```
Wire up the models:
In `article.ex`:

```elixir

  schema "articles" do
    field :title, :string
    field :url, :string
    field :categories, :string
    belongs_to :account, ArticleTrackerHd.Account # <- New line

    timestamps
  end

```

And in `account.ex`:

```elixir
schema "accounts" do
  field :email, :string
  field :password_digest, :string
  field :password, :string, virtual: true
  has_many :articles, ArticleTrackerHd.Article # <- New Line

  timestamps
end
```

Then in our ArticleController, we need to find the user and create an associated:

```elixir
def create(conn, %{"article" => article_params}) do
  account = Guardian.Plug.current_resource(conn) # <- New line
  article = Ecto.build_assoc(account, :articles) # <- New line
  changeset = Article.changeset(article, article_params) # <- New line
  # ...

end
```

Try it one more time

```bash
curl --data "article[url]=http://example.com&article[title]=A+Great+one&article[categories]=foo.bar.baz" http://localhost:4000/api/articles --header "Authorization: Bearer eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJBY2NvdW50OjEiLCJleHAiOjE0NjY2NTQ2NzUsImlhdCI6MTQ2NDA2MjY3NSwiaXNzIjoiQXJ0aWNsZVRyYWNrZXJIZCIsImp0aSI6IjBjZDNiYWZhLTY4MjYtNGJlNi1hYWM3LTMzZWJkNTAwMzM5NyIsInBlbSI6e30sInN1YiI6IkFjY291bnQ6MSIsInR5cCI6InRva2VuIn0.S3jQkmtia-mlAv0O0u1QZxWr3MFH5IBnG1C_u0Uyjq6-DOA5is3l8tKiI0M83Vfw5ADUi55uXfoRqYRY7EQ-DQ"
```

And now it's all associated.

![getinsertpic.com](http://media0.giphy.com/media/3puXxCB93ysGA/200.gif)

# Conclusion
And that's it! If you're thinking, "MAN THAT WAS NUTS!", I agree. A lot of the documentation is hard to find. I'm hoping to see more posts from YOU dear reader. Together we can make working with Phoenix and Elixir awesome.

I hope you find this useful when building Phoenix APIs! If you want to checkout the finished code, [checkout the repo](https://github.com/StevenNunez/article_tracker_hd/tree/authentication-blog-finish).

Found this interesting? Be sure to share it!

<a href="https://twitter.com/share" class="twitter-share-button" data-text="User Auth with JWT and Phoenix" data-via="_StevenNunez" data-size="large">Tweet</a>

<script>!function(d,s,id){var js,fjs=d.getElementsByTagName(s)[0],p=/^http:/.test(d.location)?'http':'https';if(!d.getElementById(id)){js=d.createElement(s);js.id=id;js.src=p+'://platform.twitter.com/widgets.js';fjs.parentNode.insertBefore(js,fjs);}}(document, 'script', 'twitter-wjs');</script>
