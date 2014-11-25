---
title: Next generation is better. I promise
date: 2014-11-25 01:13 UTC
tags: javascript, promises, generators
---

# Yey!....?
In addition to my regular hostility towards programming, I occasionally get really excited about stuff. When I heard that [Chrome 39](http://blog.chromium.org/2014/10/chrome-39-beta-js-generators-animation.html) had native support for Generators, I got really excited... until I realized I didn't really know what that meant.


## Everything but the kitchen sync
[ES6 Features](git.io/es6features) are sneaking into [browsers](http://kangax.github.io/compat-table/es6/) already. A lot of features are around dealing with asynchronous code. Generators are really cool in that they let you pause a functions execution until all `yield` statements are done. Take a look at this code:


``` javascript
function* rainbow(){
  yield "Red";
  yield "Orange";
  console.log("I'm all done")
}

var lameRainbow = rainbow();
console.log("Current Color: " + lameRainbow.next().value);
console.log("Current Color: " + lameRainbow.next().value);
console.log("Current Color: " + lameRainbow.next().value);
// ***Prints***  Current Color: Red
// ***Prints***  Current Color: Orange
// ***Prints***  I'm all done
// ***Prints***  Current Color: undefined
```

This is pretty nice! But by itself, it's kind of useless. We'll need to learn a bit more about what gets returned when we call `next`.


### NEXT!
The value we get from calling `next` is an object that has 2 properties. The value being yielded, and whether the generator is done:

``` javascript
function* rainbow(){
  yield "Red";
  yield "Orange";
  console.log("I'm all done")
}

var lameRainbow = rainbow();
lameRainbow.next();
// Object {value: "Red", done: false}
lameRainbow.next();
// Object {value: "Orange", done: false}
lameRainbow.next();
// Object {value: undefined, done: true}
```

With this in mind, it's pretty easy to go through all of the generator values in a loop:

``` javascript
function* rainbow(){
  yield "Red";
  yield "Orange";
  console.log("I'm all done")
}

var lameRainbow = rainbow();
var current = lameRainbow.next();
while(!current.done){
  console.log("Current Color: " + current.value);
  current = lameRainbow.next();
}
// ***Prints***  Current Color: Red
// ***Prints***  Current Color: Orange
// ***Prints***  I'm all done
```

### It'll be back
We saw how to pass values out of a function using `yield`, but we can also pass values back in and use it in our function. Here's an example:

``` javascript
function* brat(){
  var gift = yield "What are you giving me?";
  if(gift === "diamonds"){
    console.log("OMG YOU'RE THE BEST")
  }else{
    console.log("Whatever");
  }
}

var a = brat();
a.next(); // returns "What are you giving me?" in an object
a.next("diamonds"); // Passes "diamonds" and assigns it to our gift variable.
```

## This is going to be awesome: Promise

Another cool es6 feature is the Promise. This showed up as a way to handle asynchronous requests via the [q library](https://github.com/kriskowal/q). Other libraries have implemented this pattern, including jQuery and AngularJS and now Chrome and Firefox have native support for Promises.

### Promises Promises
A promise is a way of encapsulating how to handle asynchronous requests. The typical workflow is:

1. Make a request
2. Create a callback to handle success
3. Create a callback to handle failure

A promise wraps all of these steps together and provides you with one object you can extract the resulting code from. Here's an example of wrapping a call to the Reddit api in a promise:

``` javascript
// Assuming you have jQuery :-)
// POINT 1
var getReddit = new Promise(function(resolve, reject){
  $.getJSON('http://reddit.com/r/science.json?jsonp=?').
  success(function(result){
    // POINT 2
      resolve(result)
  }).
  error(function(){
    // POINT 3
    reject(Error("Explosion!"))})
  });

// POINT 4
getReddit.then(function(result){
     console.log(result)
   }).catch(function(error){
     console.log(error)
   });
// Gets you a reddit
```

At POINT 1 we're creating a new promise. Promises take a function that expects 2 arguments: a `resolve` function, and a `reject` function. Inside the promise, you write your normal callback laden code, dealing with success, and failure cases. With a promise, you're preparing the data, and calling the `resolve` function when it's done being processed. In our case, we aren't processing the results, so we just pass them into the `resolve` function at POINT 2. At POINT 3 we deal with errors by passing a new `Error` to the `reject` function.

By the time we get to POINT 4, we have a function that has access to the result of the call to Reddit. We can access the success case by tagging on the `then` function with instructions on what to do with the result. Nice! Errors are available by calling the `catch` function.

## When your powers combine...

Ok, let's cook. Using promises and generators we can make code that is asynchronous but looks synchronous because we encapsulate all of async stuff.  We're going to build out a few functions, and use them to make some bad ass stuff.

We'll be using a call to the Reddit api in our example. We'll set up a promise that makes a call the Reddit, but only returns the first 3 stories found.

``` javascript
var top3FromReddit = new Promise(function(resolve, reject){
  $.getJSON('http://reddit.com/r/science.json?jsonp=?').
  success(function(result){
      var stories = result.data.children.slice(0,3);
      resolve(stories);
  });
});
```

In the success callback, we're going through the results from Reddit and only returning an array of 3 stories.

Next, we'll create a generator that yields this promise. Yielding the promise will help us later when we create our async wrapper function later to resolve the promise.

``` javascript
function* redditGenerator(){
  var stories = yield top3FromReddit
  console.log(stories);
}
```

Now to build a small helper function that wraps our yielded promise

This was inspired by the example [here](https://www.promisejs.org/generators/).

``` javascript
function async(generatorFunction){
  var generator = generatorFunction();
  var promise = generator.next().value;
  promise.then(function(result){
    generator.next(result);
  })
}
```

Our `async` function takes a generator function.
We create a generator from the function, call `next` and `value` to get the promise yielded. We call the `then` function on our promise and pass the return value of the resolved promise BACK to our generator. This value will be set to `stories` inside the redditGenerator.

 _NOTE: This is an insanely limted example. Checkout the link above for a better solution, but this one works for us._


And finally, make our call to Reddit, wrapping the call in our `async` function.

``` javascript
async(redditGenerator)
```
This fires off the generator, makes the async call, assigns the value of `stories` to the resolved promise and prints it. Not bad JavaScript... Not bad...

# Recap

Generators are a really cool tool that lets you pause execution of javascript code, and coupled with promises can allow for the creation of expressive asynchronous code that appears synchronous.

You can play with a version of it here: <iframe width="100%" height="300" src="http://jsfiddle.net/whcp3Ls9/2/embedded/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>
As of November 25th, this is known to run on Chrome Canary.

Happy Clacking.
