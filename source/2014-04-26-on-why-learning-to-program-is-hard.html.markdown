---
title: On why learning to program is hard
date: 2014-04-26 20:22 UTC
tags: learning, beginner, components of programming
---
The first mention of 'Objects' with regard to some non-tangible thing was in [the late 1950s](http://en.wikipedia.org/wiki/Object-oriented_programming#History). Before that, 'Objects' didn't exist. Once you realize that all concepts in programming are made up, you start to feel less bad about not understanding them right away. 

# Learn all the things!

To be clear, when I say "Learn to program" I don't mean following a tutorial. Quite the opposite. Learning to program means you are able to create experiences through a combination of your own internal understanding of your programming platform, and the knowledge to find the answers through your available resources.

### FizzBuzz
Let's look at what you need to know to solve something like [Fizz Buzz](http://en.wikipedia.org/wiki/Fizz_buzz)

``` ruby
fizz_buzz = ->(number) do 
	"#{'Fizz' if (number % 3).zero?}#{'Buzz' if (number % 5).zero?}"
end

1.upto(100) do |num| 
	result = fizz_buzz.call(num) 
	puts(result.empty? ? num : result)
end
```

#### Reading
To read this code you'll need to know that:

1. This program runs top to bottom.
2. Expressions return values that can be assigned to variables.
3. Variables.
4. Iteration.
5. Things can be sent to [Standard Out](http://en.wikipedia.org/wiki/Standard_streams#Standard_output_.28stdout.29).
6. Conditionals.

#### Writing
Imagine you're instructing another developer to write this code. What things would they have to understand? 

"Assign the result of a lambda expression taking one argument to the variable `fizz_buzz`. In the lambda, create a string that will interpolate 'Fizz' if the argument is divisible by 3, and 'Buzz' if the number is divisible by 5. Ensure the string can be 'FizzBuzz' for numbers like 15. Then iterate from 1 to 100, calling the lambda with each number. If the result of the call is empty, return the number. Otherwise return the result".

Implementing this requires you know specific things about the language you're writing in. Idioms, conventions, methods, objects...
To implement this in ruby, you'll have to know:

1. How to create a lambda.
2. How to interpolate strings.
3. How to get the remainder from a number.
4. Comparison operators
5. Statement modifier conditional statements.
6. Ternary conditional statements.
7. How to iterate.

#### Creating
To come up with this solution you'll need a completely different skill. Programmatic thinking. Programmatic thinking is the ability to take a problem and break it down into smaller pieces. To illustrate this, have someone teach a robot how to make a smoothie. Assuming the robot knows only the things you tell it forces you to create very detailed instructions.

For FizzBuzz, the thought process is something like this.

1. I need something that figures out if it needs to return "Fizz", "Buzz" or "FizzBuzz". I could create a class with methods, or just a method, or I can use a lambda. Let's use a lambda!
2. I need to print the result out, let's loop up to 100 and call our lambda!

## Break it down
<iframe width="560" height="315" src="//www.youtube.com/embed/5MgBikgcWnY" frameborder="0" allowfullscreen></iframe>

Taking a problem apart and breaking it down into its components is the way to go. 

### Components of programming

When you're starting out, you know nothing. In order to build ANYTHING you'll need some of these things under your belt.
The good news is you can can get better at each component separately.

#### Programming concepts
To get a better grasp on programming concepts [watch talks](http://www.confreaks.com/). [Read code](https://twitter.com/readingcodegood). Lots of it. Here you'll learn about design patterns, techniques like recursion, concepts like encapsulation, namespacing, and testing.

#### Programmatic thinking
This one requires work. Go to [Code Wars](http://www.codewars.com). Do problems on [Project Euler](http://projecteuler.net). Do problems on [Ruby Quiz](http://rubyquiz.com/). 

#### Language specific stuff
Get good at your language. Read the docs. Try figuring out multiple ways to solve the same problem.

#### Which component does this serve?
Always be aware what you're working on is benefiting. Building software serves all 3, so when it doubt: BUILD!

## Silver lining
When it comes time for you to learn a new language, all you'll really have to learn is the language specific stuff! You'll only have to learn one leg of this crazy process!

### Note of caution

The trap seasoned developers fall prey to is mistaking language specific idioms/concept understandings for Programming Concepts. Just because you do it that way in C# doesn't mean that's how it's done in ruby!

## The road is long and hard
... but along the way you pick up a rocket powered motorbike. Then you don't care how long the road is, just how much fuel you have in the tank.

Happy Clacking.
