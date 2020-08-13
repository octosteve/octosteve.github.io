---
title: Phoenix Contexts and Crossing Boundaries
date: 2020-08-13 03:45 UTC
tags: Phoenix, Elixir, DDD, SQL
---

When the `contexts` were introduced in Phoenix I was really
excited. Baking in Domain Driven Design (DDD) concepts felt like
a good guardrail. It would make us create lines in our
code. Boundaries that would force us to get to know our business
better, and in the process become better developers.


  <img width="100%" src="https://images.unsplash.com/photo-1581854406835-db14e52aa35e?ixlib=rb-1.2.1&q=80&fm=jpg&crop=entropy&cs=tinysrgb&w=400&fit=max&ixid=eyJhcHBfaWQiOjEzNzc5N30" />
<small>Photo by <a href="https://unsplash.com/@hiro7jp?utm_source=seance&utm_medium=referral">Hiroshi Tsubono</a> on <a href="https://unsplash.com/?utm_source=seance&utm_medium=referral">Unsplash</a></small>


Fine and dandy until you bump up against your business'
"God" entity. The thing that is literally all things to
everyone. Working at [Flatiron School](https://flatironschool.com/) you can imagine what
our big entity in the sky is.

<img width="100%" src="https://i.imgur.com/VXixkkt.png" />

Students are everywhere. Well, sort of. When we're trying
to find the right fit for them, they're applicants. Before
that, they're leads. When they join, they're students, and
then on to becoming graduates! Then they become Job Seekers, and
then awesome developers.

They're a lot of things to different departments, and those
departments care about different parts of the student.
Admissions officers care about how to get in contact with a
student, their schedules, and program of interest. They're
interested in how applicants will handle tuition,
and finding relevant scholarship awards.

Our instructors don't care how you got there. If you're
here, it's time to work. They need your GitHub username, a
slack username, and your name.

Our career services team cares about the program you've
completed, your interests as a developer, and where they
can help best connect you with the wider developer network.

What's this look like in the context of.... contexts?

##Buckets of functionality

For your first pass you might try something like this:

<div class="mermaid">
graph BT
    c1((Accounts))
    c2([Graduate Services])
    c3([Admissions])
    c4([Education])
    r1(Web Request) \-->|Get Graduation Date| c2
    r2(Worker) \-->|Calculate Financial Clearance| c3
    r3(Web Request) \-->|Get Student Progress| c4
    r4(Web Request) \-->|Update Profile| c1
c4 \-->|get_account/1|c1
c3\-->|get_account/1|c1
c2 \-->|get_account/1| c1

</div>

You'd have all of your contexts just reach in and get the
_generic_ user data, then dump the data you don't need.
You'd then run joins on your data and attach your
transformed user data at the last minute before responding
to your requests.



<img width="100%" src="https://images.unsplash.com/photo-1589985270958-af9812c8ddb5?ixlib=rb-1.2.1&q=80&fm=jpg&crop=entropy&cs=tinysrgb&w=400&fit=max&ixid=eyJhcHBfaWQiOjEzNzc5N30" />
<small>Photo by <a href="https://unsplash.com/@sxtcxtc?utm_source=seance&utm_medium=referral">Sorin Gheorghita</a> on <a href="https://unsplash.com/?utm_source=seance&utm_medium=referral">Unsplash</a></small>


What this diagram is showing is that there's __ONE__ source
of truth for account information. None of these other
contexts can _directly_ assert FACTS regarding account
information. They can _use_ it to deliver an experience,
but it's not theirs. Any changes have to go _through_ `Accounts`

To be honest, I hate this. I either have to make clunky
requests for more data than I need, or I wind up making
functions that require a slew of Graphql like query
parameters to refine a query to what I want. All the while,
my `Accounts` context gets more and more bloated under the
weight of having to support all of these "different things
to different people" cases. This sort or reminded me of something
else though...

### Are we doing microservies?
On the surface, this approach sort of feels like an
approach to microservices where you make synchronous
requests at run time to fulfill requests sent to your
system. The same problems apply here. You need a user?
Well you've got to go to the User service. Want a course
catalog? Hit the Course Management system. Oh, it's not the data
you needed? Too bad, that's what the service gives you. This approach
always bothered me since it made YOUR system vulnerable to
another systems availability. Now you're on the hook for
circuit breakers and graceful degradation. FUN!



<img width="100%" src="https://images.unsplash.com/photo-1493836512294-502baa1986e2?ixlib=rb-1.2.1&q=80&fm=jpg&crop=entropy&cs=tinysrgb&w=400&fit=max&ixid=eyJhcHBfaWQiOjEzNzc5N30" />
<small>Photo by <a href="https://unsplash.com/@tjump?utm_source=seance&utm_medium=referral">Nik Shuliahin</a> on <a href="https://unsplash.com/?utm_source=seance&utm_medium=referral">Unsplash</a></small>


There's _another_ way of doing Microservices that might
work as good inspiration for us when working with Phoenix
Contexts.

## CQRS
If you haven't heard of CQRS, it's a method of organizing a
distributed system where the central means of communication
is messaging. Here's a sample interaction.



<div class="mermaid">
graph TB
  r1[Web Request]
 s1((Account Service))
s2((Education Service))
mb[(Message Bus)]
r1 \-->|1. Request Profile Update| s1
s1 \-->|2. CQRS Magic| s1
s1 \-->|3.Response| r1
s1 \-->|4. Emit a message| mb
mb \-->|5. Message Consumed and integrated| s2


</div>

I glossed over the CQRS magic because the thing we care
about here is the messaging as a result of a change.
Let's see what's happening in the education service.


<div class="mermaid">
graph TD
  Message \-->|Handles incoming messages| ag([Aggregator])
  ag \-->|Processes message and updates read model| db[(Read Model)]


</div>

From this point on, anytime the Education service needs
user information, it can read it from its eventually
consistent LOCAL data source. The beauty of this is that at
any point, our Account service goes down, our instructors
are able to keep on chuckin'. Here a read model is storing
ONLY the bits you need to do the job. Don't be afraid to
take in data that would be `join`ed in different databases
when writing one of these. You're optimizing for reading,
not writing.

## Ok, what's this got to do with Phoenix?
This isn't a bait and switch for me to write about CQRS. It
ties into Phoenix Contexts quite nicely. Say we look at
all of our Contexts as if they were microservices
using CQRS to keep "copies" of data. You wouldn't want to
use Phoenix Pubsub to handle internal messages to keep
these copies up to date. That'd be crazy. What you DO want
is a way to keep your underlying "local" data up to date
with what the source of truth says is FACT. You also
want to _view_ this data in the way that makes the most
sense for your domain.


<img width="100%" src="https://i.imgur.com/mKd8dvU.png" />

## Introducing SQL Views
For all intents and purposes you interact with SQL views
the same way you do any other table. You can create an Ecto
schema just like you would with any other table. You can
filter it like any other table. From your application's
perspective, it's just a table.

Let's look at some code.

## Accounts and Education
We're going to model a couple of contexts. Accounts has an
underlying Account schema, SlackAccount schema, and a
GithubAccount schema. Then we'll make the Education
Context. It's world revolves around student progress and
cohort membership. You'll see that the representation of
an `Account` doesn't really map on to what we'd need for a
student. The data is _there_, it's just not easy to get at.


<div class="mermaid">
erDiagram
    accounts ||\--o| slack-accounts : belongs-to
    accounts ||\--o| github-accounts : belongs-to
</div>

On sign up we create an `account` and you can link
a Github and Slack account if you want. This is what the
Account Schema Looks like.


<script src="https://gist.github.com/StevenNunez/13fd0ffdaf8cab02030b43ff2d618e4f.js"></script>

Here's SlackAccount


<script src="https://gist.github.com/StevenNunez/5151a3c18ca9895520b514e08758166f.js"></script>

And GithubAccount


<script src="https://gist.github.com/StevenNunez/3136237f9864024dc607b01739d4a8df.js"></script>

Let's add a top level `get/1` function to our context that
fetches WAY too much data.


<script src="https://gist.github.com/StevenNunez/4723cb5640e7ab75a22a151b345ffda2.js"></script>

So... Even looking at this, we're probably going to have to
do a lot of work to make this data useful. Checkout what
this get query looks like.



<div>
<img width="100%" src="https://i.imgur.com/S9AOptE.gif" />
<div>

While having all of this data as joins make sense in this
context, it's a pain in the neck for me to work with since
now I have to manage nested relationships every time I need
to render the parts of this account relevant to my domain.
Everytime I want to load a student's progress, I'll have to unwind
and wade through data I just don't care about. Let's leverage some
SQL to if it can help.

### Generating a view
We're going to generate a view that returns a single record
for an account, and only with the parts we care about.
SQL TIME


<div>
  <img width="100%" src="https://images.unsplash.com/photo-1512621387945-efb0d554f388?ixlib=rb-1.2.1&q=80&fm=jpg&crop=entropy&cs=tinysrgb&w=400&fit=max&ixid=eyJhcHBfaWQiOjEzNzc5N30" />
<small>Photo by <a href="https://unsplash.com/@gohrhyyan?utm_source=seance&utm_medium=referral">Goh Rhy Yan</a> on <a href="https://unsplash.com/?utm_source=seance&utm_medium=referral">Unsplash</a></small>
<div>


Let's perfect our query first. We're going to bring in all
of the tables and join them, ensuring we always return an
account if it exists, even if it's incomplete.


<script src="https://gist.github.com/StevenNunez/e50c8c0c996ae15bf9b011a4e26f2738.js"></script>

We're using `LEFT OUTER JOIN`s to prevent filtering out
accounts that don't have github  or slack accounts linked.



<div>
<img width="100%" src="https://i.imgur.com/OXPdER4.png" />
<div>

SO GREAT! This data is what I would expect if I was working
with a "Student". I can even rename columns to ones that make
sense for what I'm building. A field named `username` on
the `SlackAccount` struct makes total sense under accounts.
When I'm looking at this unified view, I'd like it named
slack_account.

Here's the view:


<script src="https://gist.github.com/StevenNunez/893bf8928ca0d8a73badd4547e296b9f.js"></script>

That's it! Now if you run a query against `students`,
you'll get this.

 id | first_name | last_name |    username    |   slack_username   | github_username
------------+------------+-----------+----------------+--------------------+------------------
 3     | Steven     | Nunez     | Steven The guy | Steven the chatter | Steven The Coder

Ok, ok... I see you squinting your eyes looking at the words
"small_hack" and wondering if I'm stealing all of your BitCoin with
this blog post. Rest assured, I have your best interest at
heart. You see, views have this weird feature in that in
most cases, you can write to them. Meaning you'd write to
the underlying table which means WE'VE BROKEN OUR CONTRACT
WITH THE SOURCE OF TRUTH. Wouldn't it be nice if we could
make this query Read only? Well, that little `WITH` trick
makes it so we spook Postgres enough to prevent it from
letting us run an insert.
Let's generate the Student Schema and run a query. Then
we'll try to insert a record.

<script src="https://gist.github.com/StevenNunez/41eb1ab7cf228895c3f4b75289a78c43.js"></script>

Let's take it for a spin.



<img width="100%" src="https://i.imgur.com/Gi4v9bb.gif" />

It's beautiful! From here you can treat this view like any
other table. You can join it against other "local" data,
and always know your data is backed by the real source of
truth.

## Odds and ends
So, this is cool. You're going to change all of your
contexts to use views now. There are a couple of things to
think about. One is that you're now tied to the source of
truth and any changes to that table's structure can break
your views! Not great. It's the reason you don't want
microservices that share the same database. One benefit we
have on our side is that our unit tests will fail our build
since missing a column would likely be catastrophic to your
app.



  <img width="100%" src="https://images.unsplash.com/photo-1434030216411-0b793f4b4173?ixlib=rb-1.2.1&q=80&fm=jpg&crop=entropy&cs=tinysrgb&w=400&fit=max&ixid=eyJhcHBfaWQiOjEzNzc5N30" />
<small>Photo by <a href="https://unsplash.com/@craftedbygc?utm_source=seance&utm_medium=referral">Green Chameleon</a> on <a href="https://unsplash.com/?utm_source=seance&utm_medium=referral">Unsplash</a></small>


I wanted to mention _Materialized Views_. We're using
regular old views which means that when you run the query,
it's running the underlying query RIGHT THEN AND THERE.
This means you'll always have the latest data available. So
what's a _Materialized View_? It's a view that's stored on
disk and is rerun based on some trigger, say a record was
inserted into a specific table. Why would you want this?
What do you lose? If your queries are slow, Materialized
Views get you a wicked fast response since it's reading a
cached value. What you lose is your data might lag a bit
depending on how expensive the query is. Setup is a little
different, but the concept is the same.

That's all for today. Thanks for reading!
