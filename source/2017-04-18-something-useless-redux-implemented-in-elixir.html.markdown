---
title: Something Useless - Redux Implemented in Elixir
date: 2017-04-18 15:38 UTC
tags: elixir, redux
---

# Something completely useless - Redux Implemented in Elixir
I got a chance to teach [Redux](http://redux.js.org/) recently and what a time! For something so simple, it sure can get complicated. If you take a look at my [previous articles](2014/04/13/closures.html) you might notice a technique I use to get my head around concepts. Bring that bad boy on to my own turf.

![](https://media.giphy.com/media/OhQBBFi64Z81a/giphy.gif)

A naïve implementation of Redux using JavaScript might look something like this:

``` javascript
const initializerAction = {type: "@@INIT"}
const createStore = (reducer, state=undefined) => {
state = reducer(state, initializerAction)
let subscribers = []
  return {
    getState() {
      return state
    },
    subscribe(subscriber){
      subscribers.push(subscriber)
      return () => {
        subscribers = subscribers.filter(s => s !== subscriber)
      }
    },
    dispatch(action){
      state = reducer(state, action)
      subscribers.forEach(s => s())
    }
  }
}

const countReducer = (state=0, action) => {
  switch(action.type){
    case "INCREMENT":
      return state + 1
    default:
      return state
  }
}
const store = createStore(countReducer)
const removeSubscriber = store.subscribe(() => console.log("I got called", store.getState()))
store.dispatch({type: "INCREMENT"})
// Should log "I got called: 1"

removeSubscriber()
store.dispatch({type: "INCREMENT"})
// Should log nothing"
```

Cool. So we can build out a store that calls a single reducer. The store can notify subscribers of changes. We'll work on implementing this in Elixir, and we'll even handle the case with combined reducers. Along the way, we'll see where the JavaScript implementation falters, blocking execution of independent reducers and how to get around that in Elixir.

# Let's dream
The code of my dreams looks something like:

``` elixir
{:ok, store} = Store.start_link(CountReducer)

subscriber_ref = Store.subscribe(store, fn (state) ->
	IO.puts "I Got called: #{state}"
end)

Store.dispatch(store, %{type: "INCREMENT"})
# Should log "I got called: 1"

Store.remove_subscriber(store, subscriber_ref)
Store.dispatch(store, %{type: "INCREMENT"})
# Should log nothing"
```

We swapped out a few things. This is Elixir so we're passing in a reference to the `Store` pid on every invocation. We also changed how we unsubscribe from a store, returning a reference value instead of a function that magically calls in the right scope. Also, we pass the current state to the subscribers on every call. This looks great, but how do we build it?

## Enter the GenServer
I *love* the Erlang lords for creating the GenServer. It works for EVERYTHING.
Let's start by implementing the `get_state` function on a `Store` process.

``` elixir
defmodule Store do
  @initializer_action %{type: "@@INIT"}

  # Code your Mom calls
  def start_link(reducer, initial_state \\ nil) do
    GenServer.start_link(__MODULE__, [reducer, initial_state])
  end

  def get_state(store) do
    GenServer.call(store, {:get_state})
  end

  # Code Joe Armstrong Calls
  def init([reducer, initial_state]) do
    store_state = apply(reducer, :reduce, [initial_state, @initializer_action])
    {:ok, %{reducer: reducer, store_state: store_state}}
  end

  def handle_call({:get_state}, _from, state) do
    {:reply, Map.get(state, :store_state), state}
  end
end
```

The code above will use the `apply` function to dispatch a message to a module. If our reducer is a module named `CountReducer`, it would call `CountReducer.reduce(state, action)`. Pretty nice.

The reducer will be a module with a `reduce/2` function. Reducers have a few responsibilities.
* They provide default values for state on initialization
* They return a new version of state based on an action being passed
* They ignore messages it doesn't care for

The Reducers don't care what the state is, all they do is get state, and an action, then respond with new state. Here's a sample reducer:

``` elixir
defmodule Reducer do
  @callback reduce(any, Map :: map()) :: any
end

defmodule CountReducer do
  @behaviour Reducer
  def reduce(nil, action), do: reduce(0, action)
  def reduce(state, action), do: do_reduce(state, action)

  defp do_reduce(state, %{type: "INCREMENT"}), do: state + 1
  defp do_reduce(state, _), do: state
end
```

We first create a `behaviour` to ensure anyone that plays the `Reducer` role knows how to do the job. We need it to have a `reduce/2` function to support the `Store` module's expectation. The rest is just some good old fashioned PATTERN MATCHING. The first 2 `reduce/2` heads are just clean up. If we get a nil value, then just call it again with a 0, our `do_reduce/2` functions are where the actual reducing happens.

## Time to run it

Throw all of that in a file named `e_dux.exs` a open up `iex`.

``` elixir
c "e_dux.exs"

{:ok, store} = Store.start_link(CountReducer)
Store.get_state(store) # => 0
```

Not too exciting. Let's add the ability to `dispatch` an action. Back in our `Store`:

``` elixir
defmodule Store do
  # Code for your Mom
  # Existing Mom code...
  def dispatch(store, action) do
    GenServer.cast(store, {:dispatch, action})
  end

  # Code for Joe Armstrong
  # Existing Joe code...
  def handle_cast({:dispatch, action}, %{reducer: reducer, store_state: store_state} = state) do
    store_state = apply(reducer, :reduce, [store_state, action])
    {:noreply, Map.put(state, :store_state, store_state)}
  end
```

Here we cast asynchronously using `GenServer.cast/2`. This strategy makes it so our dispatch code won't block our calling code. Imagine if a reducer took a long time to finish. Or better yet, imagine if we have to run through several reducers? This not blocking takes advantage of Elixir's real power: Concurrency. Try this out by loading the file like before and run this:
```elixir
c "e_dux.exs"

{:ok, store} = Store.start_link(CountReducer)
Store.get_state(store) # => 0
Store.dispatch(store, %{type: "INCREMENT"})
Store.get_state(store) # => 1
```

WOOT! Got some `INCREMENT` action! Let's move on to adding subscribers and notifying them.

## I'd like to subscribe to your newsletter
Back to our dream code:
``` elixir
{:ok, store} = Store.start_link(CountReducer)

subscriber_ref = Store.subscribe(store, fn (state) ->
	IO.puts "I Got called: #{state}"
end)

Store.dispatch(store, %{type: "INCREMENT"})
# Should log "I got called: 1"

Store.remove_subscriber(store, subscriber_ref)
Store.dispatch(store, %{type: "INCREMENT"})
# Should log nothing"
```

We want a `dispatch` to notify our existing subscribers after we've updated our state. We'll also have our `store` track its subscribers. We'll be updating `init/1`, as well as adding `subscribe/2` and `remove_subscriber/2`

### init/1

``` elixir
def init([reducer, initial_state]) do
  store_state = apply(reducer, :reduce, [initial_state, @initializer_action])
  {:ok, %{reducer: reducer, store_state: store_state, subscribers: %{}}} # add new subscribers map to state
end
```

Here we add a new key to our state map: subscribers. We'll be taking advantage of this being a map when we delete subscribers in `remove_subscriber/2`

### subscribe/2 remove_subscriber/2 and call backs

``` elixir
# For mama
def subscribe(store, subscriber) do
  GenServer.call(store, {:subscribe, subscriber})
end

def remove_subscriber(store, ref) do
  GenServer.cast(store, {:remove_subscriber, ref})
end

# For Joey
def handle_call({:subscribe, subscriber}, _from, %{subscribers: subscribers} = state) do
  ref = make_ref()
  {:reply, ref, put_in(state, [:subscribers, ref], subscriber)}
end

def handle_cast({:remove_subscriber, ref}, %{subscribers: subscribers} = state) do
  subscribers = Map.delete(subscribers, ref)
  {:noreply, Map.put(state, :subscribers, subscribers)}
end
```
Our `subscribe/2` function will add our subscriber to our map, storing a `ref` as the key. Since a ref is a unique value, it's a great way to well... reference stuff. We'll return that `ref` to our caller. `remove_subscriber/2` takes a `store` pid and a `ref`. This updates our `subscriber` map. This is done in a `cast` since we don't care about the return value.

### dispatch/2
```elixir
def handle_cast({:dispatch, action}, %{reducer: reducer, store_state: store_state, subscribers: subscribers} = state) do
  store_state = apply(reducer, :reduce, [store_state, action])
  for {_ref, sub} <- subscribers, do: sub.(store_state) # notify those subscribers
  {:noreply, Map.put(state, :store_state, store_state)}
end
```
FINALLY we update `handle_cast/2` for our `dispatch/2` function to call every subscriber.

With this code, our dreams are ALIVE!
```elixir
{:ok, store} = Store.start_link(CountReducer)

subscriber_ref = Store.subscribe(store, fn (state) ->
	IO.puts "I Got called: #{state}"
end)

Store.dispatch(store, %{type: "INCREMENT"})
# Should log "I got called: 1"

Store.remove_subscriber(store, subscriber_ref)
Store.dispatch(store, %{type: "INCREMENT"})
# Should log nothing"
```

## Combined Reducers
So Redux lets you break down complex logic into multiple reducers. Something like:

``` javascript
const rootReducer = combineReducers({
  count: countReducer,
  square: squareReducer
})

const store = createStore(rootReducer)
store.getState() // {count: 0, square: 2} // default to 2 so we can get some good squaring action
store.dispatch({type: "INCREMENT"})
store.getState() // {count: 1, square: 4}
```

Internally all a combined reducer is is a reducer that delegates to other reducers. More naive code:

```javascript
const combineReducers = (reducerMap) => {
  return (state, action) => {
    return Object.keys(reducerMap).reduce((map, stateName) => {
      map[stateName] = reducerMap[stateName](state[stateName], action)
      return map
    }, {})  
  }
}

const countReducer = (state=0, action) => {
  switch (action.type) {
    case "INCREMENT":
      return state + 1
    default:
      return state
  }
}

const squareReducer = (state=2, action) => {
  switch (action.type) {
    case "INCREMENT":
      return state * state
    default:
      return state
  }
}

const rootReducer = combineReducers({count: countReducer, square: squareReducer})
rootReducer({count: 0, square: 2}, {type: "INCREMENT"}) // {count: 1, square: 4}
```

Here we return a function that when called with a state and action will delegate portions of the state to specific reducers. Pretty nice! But something wicked lies in this code. When our reducer dispatches, it can only delegate to one reducer at a time, even though their state is completely isolated. This would be a perfect time for some concurrency! Let's look at how we can handle this in Elixir.

### CombineReducer Module
We'll start out with a change to how treat individual reducers vs a Map with reducers. Above our existing `init/1` function, add this:

```elixir
# New stuff!
def init([reducer_map, nil]) when is_map(reducer_map), do: init([reducer_map, %{}])
def init([reducer_map, initial_state]) when is_map(reducer_map) do
  store_state = CombineReducers.reduce(reducer_map, initial_state, @initializer_action)
  {:ok, %{reducer: reducer_map, store_state: store_state, subscribers: %{}}} # add new subscribers map to state
end

# Crusty old code
def init([reducer, initial_state]) do
  store_state = apply(reducer, :reduce, [initial_state, @initializer_action])
  {:ok, %{reducer: reducer, store_state: store_state, subscribers: %{}}} # add new subscribers map to state
end
```

Converting nil values to an empty map will let our reducers receive `nil` as their state when we call something like `state[:count]`. They'll then convert that to their expected default state.

One small change to `dispatch/2` to support our reducer potentially being a map instead of a module:

```elixir
# New hotness
def handle_cast({:dispatch, action}, %{reducer: reducer_map, store_state: store_state, subscribers: subscribers} = state) when is_map(reducer_map) do
  store_state = CombineReducers.reduce(reducer_map, store_state, action)
  for {_ref, sub} <- subscribers, do: sub.(store_state) # notify those subscribers
  {:noreply, Map.put(state, :store_state, store_state)}
end

# Old and crust
def handle_cast({:dispatch, action}, %{reducer: reducer, store_state: store_state, subscribers: subscribers} = state) do
  store_state = apply(reducer, :reduce, [store_state, action])
  for {_ref, sub} <- subscribers, do: sub.(store_state) # notify those subscribers
  {:noreply, Map.put(state, :store_state, store_state)}
end
```

And finally, our brand new module a brand new module to handle our CombineReducers logic.
``` elixir
defmodule CombineReducers do
  def reduce(reducers, state, action) do
    for {state_name, reducer} <- reducers do
      Task.async(fn () ->
        {state_name, apply(reducer, :reduce, [state[state_name], action])}
      end)
    end
    |> Enum.map(&Task.await/1)
    |> Enum.into(%{})
  end
end
```

Ok, dopeness alert. Notice how we're using a task to run this calculation. This means that the calculation for ALL of the reducers will take as long as the slowest one, not the culmination of all of their time. THIS IS REALLY COOL. How are you not excited?!

Let's test it out. First create a `SquareReducer`:
```elixir
defmodule SquareReducer do
  @behaviour Reducer
  def reduce(nil, action), do: reduce(2, action)
  def reduce(state, action), do: do_reduce(state, action)

  defp do_reduce(state, %{type: "INCREMENT"}), do: state * 1
  defp do_reduce(state, _), do: state
end
```

Then Run this!

```elixir
{:ok, store} = Store.start_link(%{count: CountReducer, square: SquareReducer})

Store.dispatch(store, %{type: "INCREMENT"})
Store.get_state(store)
```

SO COOL!!!

# Closing up
This bit of hacking really deepend my understanding of Redux, how combined reducers worked, and what their limitations where. I also got a chance to work on some interesting Elixir. I don't really have a use for this pattern. It seems like it might be stepping on some of the same pain points you'd solve with [GenEvent](https://hexdocs.pm/elixir/GenEvent.html). Thanks for reading.

Want to see more of something mentioned here? Leave any ideas in the comments below.

Happy Clacking.
