---
title: Something Useless - Redux Implemented in Elixir
date: 2017-04-18 15:38 UTC
tags:
---

# Something completely useless - Redux Implemented in Elixir
I got a chance to teach [Redux](http://redux.js.org/) recently and what a time! For something so simple, it sure can get complicated. If you take a look at my [previous articles](2014/04/13/closures.html) you might notice a technique I use to get my head around concepts. Bring that bad boy on to my own turf.

![](https://media.giphy.com/media/OhQBBFi64Z81a/giphy.gif)

A naïve implementation of redux using JavaScript might look something like this:

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

const myReducer = (state=0, action) => {
  switch(action.type){
    case "INCREMENT":
      return state + 1
    default:
      return state
  }
}
const store = createStore(myReducer)
const removeSubscriber = store.subscribe(() => console.log("I got called", store.getState()))
store.dispatch({type: "INCREMENT"})
// Should log "I got called: 1"

removeSubscriber()
store.dispatch({type: "INCREMENT"})
// Should log nothing"
```

Cool. So we can build out a store that calls a single reducer and that returns a value. We'll work on implementing this in Elixir, and we'll even handle the case with combined reducers. Along the way, we'll see where the JavaScript implementation falters, blocking execution of independent actions and how to get around that in Elixir. Enough talk!


# Let's dream
The code of my dreams looks something like:

``` elixir
{:ok, store} = Store.start_link(MyReducer)

subscriber_ref = Store.subscribe(store, fn (state) ->
	IO.puts "I Got called: #{state}"
end)

Store.dispatch(store, %{type: "INCREMENT"})
# Should log "I got called: 1"

Store.remove_subscriber(store, subscriber_ref)
Store.dispatch(store, %{type: "INCREMENT"})
# Should log nothing"
```

We swapped out a few things. This is Elixir so we're passing in a reference to the Store pid on every invocation. We also changed how we unsubscribe from a store, returning a reference value instead of a function that magically calls in the right scope. That would have worked but felt gross. Also, we pass the current state to the subscribers on every call. This looks great, but how do we build it?

## Enter the GenServer
I love the Erlang lords for creating the GenServer. It works EVERYWHERE.
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

The code above will use the `apply` function to dispatch a message to a dynamic module, So if our reducer is a module named `MyReducer`, it would call `MyReducer.reduce(state, action)`. Pretty nice.

The reducer will be a module with a `reduce/2` function.

``` elixir
defmodule Reducer do
  @callback reduce(any, Map :: map()) :: any
end

defmodule MyReducer do
  @behaviour Reducer
  def reduce(nil, action), do: reduce(0, action)
  def reduce(state, action), do: do_reduce(state, action)

  defp do_reduce(state, %{type: "INCREMENT"}), do: state + 1
  defp do_reduce(state, _), do: state
end
```

We first create a `behaviour` to ensure anyone that plays the `Reducer` role knows how to do the job. We need a it to have that `reduce/2` function, so we apply it to `MyReducer`. The rest is just the reason you fell in love with Elixir. PATTERN MATCHING. The first 2 reduce heads are just clean up. If we get a nil value, then just call it again with a 0, our `do_reduce/2` functions are where the actual reducing happens.

## Time to run it

Throw all of that in a file named `e_dux.exs` a open up `iex`.

``` elixir
c "e_dux.exs"

{:ok, store} = Store.start_link(MyReducer)
Store.get_state(store) # => 0
```

Not too exciting. Let's add the ability to `dispatch` an action. Back in our `Store`:

``` elixir
defmodule Store do
  # Code for your Mom
  def dispatch(store, action) do
    GenServer.cast(store, {:dispatch, action})
  end

  # Code for Joe Armstrong
  def handle_cast({:dispatch, action}, %{reducer: reducer, store_state: store_state} = state) do
    store_state = apply(reducer, :reduce, [store_state, action])
    {:noreply, Map.put(state, :store_state, store_state)}
  end
```

Here we cast asynchronously using `Genserver.cast/2`, since we don't care about the return value. Try this out by loading the file like before and run this:
```elixir
c "e_dux.exs"

{:ok, store} = Store.start_link(MyReducer)
Store.get_state(store) # => 0
Store.dispatch(store, %{type: "INCREMENT"})
Store.get_state(store) # => 1
```

WOOT! Got some `INCREMENT` action! Let's move on to adding subscribers and notifying them.

## I'd like to subscribe to your newsletter
Right from our dream code:
``` elixir
{:ok, store} = Store.start_link(MyReducer)

subscriber_ref = Store.subscribe(store, fn (state) ->
	IO.puts "I Got called: #{state}"
end)

Store.dispatch(store, %{type: "INCREMENT"})
# Should log "I got called: 1"

Store.remove_subscriber(store, subscriber_ref)
Store.dispatch(store, %{type: "INCREMENT"})
# Should log nothing"
```

We want a `dispatch` to notify our existing subscribers after we've updated our state. We'll also have to have our `store` track its listeners. We'll be updating `init/1`, as well as adding `subscribe/2` and `remove_subscriber/2`

### init/1
``` elixir
def init([reducer, initial_state]) do
  store_state = apply(reducer, :reduce, [initial_state, @initializer_action])
  {:ok, %{reducer: reducer, store_state: store_state, subscribers: %{}}} # add new subscribers map to state
end
```

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
  subscribers = Map.put(subscribers, ref, subscriber)
  {:reply, ref, Map.put(state, :subscribers, subscribers)}
end

def handle_cast({:remove_subscriber, ref}, %{subscribers: subscribers} = state) do
  subscribers = Map.delete(subscribers, ref)
  {:noreply, Map.put(state, :subscribers, subscribers)}
end
```
### dispatch/2
```elixir
def handle_cast({:dispatch, action}, %{reducer: reducer, store_state: store_state, subscribers: subscribers} = state) do
  store_state = apply(reducer, :reduce, [store_state, action])
  for sub <- Map.values(subscribers), do: sub.(store_state) # notify those subscribers
  {:noreply, Map.put(state, :store_state, store_state)}
end
```

WHEW! `subscribe/2` takes an anonymous function and stores it. Later, when you call `dispatch/2` it will iterate over the collected subscribers and notify them of the current state.

With this code, our dreams are ALIVE!
```elixir
{:ok, store} = Store.start_link(MyReducer)

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
const rootReducer = combinedReducers({
  count: countReducer,
  letter: letterReducer
})

const store = createStore(rootReducer)
store.getState() // {count: 0, letter: "a"}
```
