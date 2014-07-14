---
title: Using Ruby to learn JavaScript
date: 2014-05-18 20:16 UTC
tags: ruby, javascript, polyglot
---

I've written about why I believe [learning to program is hard](2014-04-26-on-why-learning-to-program-is-hard.html). If you've made it to the other side, rejoice! It's time for the easy part. Learning a new language. Let's see how knowing Ruby can help us learn a little JavaScript. (I'm assuming you know Ruby for the duration of this post)

# Ruby to JavaScript
First, let's find some similarities.
Both are:

* Dynamically Typed
* Both are Object Oriented
* Functions are first class citizens

Let's look at these first

## Dynamically Typed

``` ruby
my_account = {}
my_account = []
```
vs

``` javascript
var myAccount = {};
myAccount = [];
```

With these 2 lines of code, we can start to identify some differences between the languages.

### The `var` keyword
Variables in JavaScript __must__ be declared. You __must__ communicate this intention, otherwise weird things start to happen. The following code works, but not like you think.

Run this in Developer Tools

``` javascript
var varTest = function(){
  var totallyFine = 17;
  ohMy = 100;
}
varTest();

console.log(ohMy); // 100
console.log(totallyFine); // explosion!  
```
Why's this a big deal? This means that if any code, anywhere creates a variable in a function without the `var` keyword, it's with you for the rest of your app. Other things can change it. For a real life case of this biting someone, checkout [this  post](http://blog.safeshepherd.com/23/how-one-missing-var-ruined-our-launch/).

Strict mode prevents this tomfoolery.

``` javascript
var varTest = function(){
  'use strict';
  var totallyFine = 17;
  ohMy = 100;
}
varTest(); // blows up here since ohMy is not defined
```
Another difference are those semicolons! Semicolons are used to end a statement. It may be your first impulse to throw semicolons everywhere but [don't](http://www.codecademy.com/blog/78-your-guide-to-semicolons-in-javascript).
JavaScript also uses camelCase for its variables, vs Ruby's snake case.

## Object Orientation
Both languages can create structures that act as templates for instances of itself.

``` ruby
class Student
end
bob = Student.new
```

vs

``` javascript
var Student = function(){};
var bob = new Student;
```

In JavaScript, we supply what's called a Constructor Function. One that responds to the `new` keyword by returning an instance of the function. Let's flesh this out some more.

``` ruby
class Student
  attr_reader :name
  def initialize(name)
    @name = name
  end
end
bob = Student.new("Bob")
bob.name # => "Bob"
```

vs

``` javascript
var Student = function(name){
  this.name = name;
};

var bob = new Student("Bob");
bob.name; // "Bob"
```

The Constructor Function can take arguments like any other function.

To add methods to the Factory, add them to the prototype. Then all instances will have access to those methods.

``` javascript
var Student = function(name){
  this.name = name;
};

Student.prototype.grade = function(){
  if(this.name == "Steven"){
    return "A";
  }else{
    return "F";
  }
};

var bob = new Student("Bob");
bob.grade() // "F" sorry Crouton
var steven = new Student("Steven");
steven.grade() // "A" obviously
```


Inside the Constructor Function and prototype methods you have access to this mysterious `this` thing. A new `this` is created whenever you call a function with the `new` keyword. By convention, Constructor Functions start with a capital letter.

``` javascript
var ThisOrThat = function(){
  this.thang = "This thang";
}

var thisThing = new ThisOrThat();
thisThing.thang; // "This Thang"
```

There's another way to make objects in JavaScript by using an object literal.

``` javascript
var myObject = {
  value1: "I'm a value",
  action: function(){
    return "Live Action! " + this.value1 ;
  }
}

myObject.value1; // I'm a value
myObject.action(); // "Live Action! I'm a value""
```

Here, the `action` method has access to other properties in the object through `this`.

``` javascript
var myObject = {
  getThis: function(){
    return this;
  }
}

myObject === myObject.getThis(); // true
```

This is akin to creating an Object instance in Ruby and attaching singleton methods.


``` ruby
my_object = Object.new
def my_object.value1
  "I'm a value"
end

def my_object.action
  "Live Action #{self.value1}"
end

def my_object.get_self
  self
end

my_object.value1 # Same
my_object.action # As above
my_object == my_object.get_self # => true

```
### Properties in Ruby

One thing you'll notice is that Ruby has no concept of 'properties' on objects. Everything in Ruby is responding to a message sent. Nothing is stopping you from adding new properties on an existing object in JavaScript.

``` javascript
var myObject = {};
myObject.value1 = "I'm a value";
myObject.value1;
```

In Ruby, it would break.

``` ruby
my_object = Object.new
my_object.value1 = "I'm a value" # => "NoMethodError"
```

They behave more like Ruby's OpenStruct class

``` ruby
require 'ostruct'
my_object = OpenStruct.new
my_object.value1 = "I'm a value"
my_object.value1
```

## First Class Functions
What the hell does that even mean?
> Specifically, this means the language supports passing functions as arguments to other functions, returning them as the values from other functions, and assigning them to variables or storing them in data structures.

My post on [closures](2014-04-13-closures.html) shows that Ruby has passable functions in the form of lambdas, but it's used a lot more in JavaScript!!

Look at this JavaScript code that takes a function, and returns a new function based on the return value of the former function.

```javascript
var poppaFunction = function(poppa){
  return function(msg){
    console.log('Hello, ' + poppa + "! " + msg);
  }
}

var babyFunction = poppaFunction("Big Papa");
babyFunction("Have a great night!");
```
and the same in Ruby

```ruby
poppa_lambda = lambda do |poppa|
  lambda {|msg| puts "Hello #{poppa}! #{msg}"}
end

baby_lambda = poppa_lambda.call("Big Papa")
baby_lambda.call("Have a great night!")
```

One important distinction is the the differences in return values. In JavaScript, functions **must** return a value, otherwise they return `undefined`. In ruby, the last executed line of a method/lambda/block is what's returned, unless you use and explicit `return`.

# Differences

We've seen a few differences already. Some with syntax (semicolon use, variable declarations), others with the object model (Ruby only responding to messages, and JavaScript taking any old property). In this section I want to dive into 2 things, primarily how these languages represent the current object, and equality.

## `self` and `this`

Ruby's rules for what changes self are pretty straight forward.

```ruby
class RubySelf
  self # => RubySelf
  def my_instance_method
    self # => the instance
  end
end
self # => main
```

That's pretty much it. There's also [`instance_eval` and `class_eval`](http://www.jimmycuadra.com/posts/metaprogramming-ruby-class-eval-and-instance-eval). A topic for another time.

In JavaScript it's not so simple.

We've seen that `this` points to current object inside an object.
```javascript
var myObject = {
  getThis: function(){
    return this;
  }
}

myObject === myObject.getThis(); // true
```

But what about here?
``` javascript
var myFunction = function(){
  this.value = "A value";
  return this
};

var result = myFunction();
value; // "A value" We put it on the global scope!!!!
result === window // true OMG!!!
```

`this` inside a regular function points to the root object. In browser that's `window`.

Again, strict mode saves our butts here.

``` javascript
var myFunction = function(){
  'use strict';
  this.value = "A value";
  return this
};

var result = myFunction(); // TypeError: Cannot set property 'value' of undefined
```

How about this one?

``` javascript
var myObject = {
  collection: [1,2,3],
  iterate: function(){
    this.collection.forEach(function(item){
      this.collection.push(item * item); // This gon break.
    })
  }
};

myObject.collection;
myObject.iterate(); // TypeError: Cannot read property 'push' of undefined
myObject.collection;
```

This new function changed `this` back to the default `this`, the `window` in the case of the browser.

In cases like this we have to keep track of our previous self in a variable. Usually this variable is named `_that` or `self`.
```javascript
var myObject = {
  collection: [1,2,3],
  iterate: function(){
    var self = this;
    this.collection.forEach(function(item){
      self.collection.push(item * item); // This gon break.
    })
  }
};

myObject.collection;
myObject.iterate(); // TypeError: Cannot read property 'push' of undefined
myObject.collection; // [1, 2, 3, 1, 4, 9]
```

Sharp viewers know `forEach` takes an optional argument for the object to stand in as `this`.

``` javascript
this.collection.forEach(function(item){
  this.collection.push(item * item); // This gon break.
}, this) // this this is the original this not that this. :-)
```

## Equality
Ruby's equality testing has 2 forms: `==` and `===`. The `==` tests for equality as defined by the object.

```ruby
5 == 5
"Steven" == "Steven"

class MyObject
  attr_reader :description
  def initialize(description)
    @description = description
  end

  def ==(other)
    other.is_a?(MyObject) && self.description == other.description
  end
end

obj1 = MyObject.new("A plain object")
obj2 = MyObject.new("A plain object")
obj1 == obj2 # => true
```

And Case equality, used for `case` statements. In a new object, `===` delegates to `==`

``` ruby
(1..10) === 5
/even/ === "Steven"
```

JavaScript has no way to override equality on objects. You CAN use equality on primitives like strings and numbers, but here lies the danger. This is from ["JavaScript: The Good Parts"](http://www.amazon.com/JavaScript-Good-Parts-Douglas-Crockford/dp/0596517742)

``` javascript
0 == '' // true
false == 'false' // false
false == '0' // true
null == undefined // true
```

[Wat](https://www.destroyallsoftware.com/talks/wat). The `==` in JavaScript tries to coerce or convert the things being compared to be the same type, which might sound helpful, but it's not. Use `===` and `!==` to ensure you're comparing type and value.

``` javascript
0 === '' // false
false === 'false' // false
false === '0' // false
null === undefined // false as it should be!
```

Learning your first programming language is hard. I hope I've shown you that learning your second one, isn't that bad.

Happy Clacking!
