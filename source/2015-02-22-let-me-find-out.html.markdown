---
title: Let me find out
date: 2015-02-22 03:41 UTC
tags: ruby, enumerable, find
---

# Finders keepers

The `find` method is one of the first methods I suggest people use when they're suffering from __eachitis__.

>__EACHITIS__: A condition where all solutions involving collections are solved using the `each` method.

One problem with `find` is how to interpret the funky method signature.

``` ruby
find(ifnone = nil) { |obj| block } → obj or nil
```

From this signature, you might think you could just give an argument to `find` and it will be returned if no value is found. But that's when disaster strikes.

``` ruby
(1..5).find("nope"){|num| num > 9000}
# => NoMethodError: undefined method `call' for "nope":String
```

Rude! It turns out, Ruby wants a `call`able object to be passed in as an argument, which unfortunately, a string is not. The first callable object that comes to mind is the terrorizing `lambda`. Let's see what happens if we wrap our default value in a lambda instead of passing a string. We'll use the stabby lambda syntax to keep it short.

``` ruby
(1..5).find(->{"nope"}){|num| num > 9000}
# => "nope"
```

Great! You might be asking yourself, "Why the hell is this a thing?". Well reader, checkout this example to see if it makes more sense.

``` ruby
(1..5).find(->{raise "hell"}){|num| num > 9000}
# => Hell has been raised
```

Lambdas are a way of delaying execution. Here we're delaying the calling of an exception until it's necessary. You can also use this to make calls to an API only when needed.

## The wheels are turning

The beauty of Duck Typing is that all we need is an object that responds to `call`. That means we can do some pretty crazy stuff.

Take this class for example:

``` ruby
require 'json'
require 'rest-client'
class PostNotFound < StandardError; end
Post = Struct.new(:title, :url)

class Reddit
  attr_reader :subreddit, :posts
  attr_accessor :last_term

  def initialize(subreddit)
    @subreddit = subreddit
    @posts = []
  end

  def search(term, not_found=method(:retry_search))
    self.last_term = term
    posts.find(not_found){|post| post.title =~ /#{last_term}/ }
  end

  def refresh
    results = JSON.parse(RestClient.get("http://www.reddit.com/r/#{subreddit}.json"))
    posts.concat(to_posts(results["data"]["children"])).uniq
  end

  private
  def retry_search
    refresh
    search(last_term, -> {raise PostNotFound})
  end

  def to_posts(post_collection)
    post_collection.map{|post| Post.new(post["data"]["title"], post["data"]["url"])}
  end
end

client = Reddit.new('vegetarianism')
client.search("Ham")
```
Calling `search` fails, it calls the `retry_search` method. It literally
 calls the ruby `Method` object. Pretty freakin' sweet.

## Conclusion
Ruby's language features are one of the most robust I've had the pleasure to work with. This mundane method offers a nice splash of awesome by taking a callable object as its requirement.
