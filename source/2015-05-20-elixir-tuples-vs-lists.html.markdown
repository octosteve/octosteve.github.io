---
title: Elixir: Tuples vs Lists
date: 2015-05-20 15:28 UTC
tags: Elixir, Fundamentals
---

# Immutability Primer
Elixir is a Functional programming language with immutable types. You might see code like the following and think that we're mutating a value.

``` elixir
names = ["Steven", "Kelly", "Michael"]
names = ["Telly"] ++ names
names # => ["Telly", "Steven", "Kelly", "Michael"]
```
Elixir is giving us a new copy of the array. How it got this might be a bit confusing though. For starters, this is how Elixir stored that original `names` list.

``` elixir
names = ["Steven" | ["Kelly" | ["Michael"]]]
```

It stores it as a linked list, with an existing list nested inside of a new list. Elixir has some ways of pattern matching this list by breaking it into the list's `head` and `tail`.

The `head` is the first element, while the `tail` is... the rest.

``` elixir
[head| tail] = ["Steven" | ["Kelly" | ["Michael"| []]]]
head #=> "Steven"
```

When we added `"Telly"` to the list, we were creating a NEW list made up of "Telly" being the head, and the original list being the tail.

``` elixir
names = ["Steven", "Kelly", "Michael"]
names = ["Telly"] ++ names
names # => ["Telly", "Steven", "Kelly", "Michael"]
# or
# ["Telly"| ["Steven", "Kelly", "Michael"]]
```

OK, so you can't modify existing values.

## Lists vs Tuples
I was a bit confused to see that there were more than 2 types for storing lists. Coming from ruby, we have `Array`s, that's it.

Tuples are Elixir's other way of storing a list of stuff.

``` elixir
names = {"Steven", "Kelly", "Michael"}
names = Tuple.insert_at(names, 0, "Telly") #=> {"Telly", "Steven", "Kelly", "Michael"}
```

### So they're the same...?

Well... no. Why have 2 if they were?

### Lookup Showdown
Tuples are stored contiguously in memory, making operations like finding the size of a tuple constant time. Fetching an element at an index is also constant.

Finding an element at a specific position in a List is a bit more laborious. We have to unwrap each layer of the list and count every element. This makes fetching items in an array linear time.

Elixir tries to guide you to the right type by exposing the `elem` look up methods on tuples.

### Deriving a collection

Prepending to a List is fast. All you do is create a new list with the added value as the head and the rest as the tail, like our `names` example. But what about appending? Well, to add something to the end of a list, we have to rebuild the entire list, and add our new list to the end. As our list grows, we have to execute more operations to deconstruct our list.

Adding to a Tuple is also expensive since it's stored in a memory contiguously, you have to copy the entire collection and add the new element.


## Future

I hope this was useful. I want the world using Elixir!
