---
title: Custom params in route resources
date: 2014-02-15 21:09 UTC
tags: rails, ruby, routing, web
---

Anyone who knows me will tell you I'm a huge rails fanboy. There are times when you get _stuck_.
A name can't be changed because it's something the framework decided on. I ran
into this recently when working with nested routes.

# The problem
Working at [The Flatiron School](http://flatironschool.com/) has been awesome. I've gotten to work on some great projects.
While working on a Progress Dashboard I came across this code for creating nested routes. WALL OF CODE COMING.

### In my routes file
``` ruby
resources :dashboards, only: [:show, :index] do
  resources :assignments, only: [:show]
  resources :students, only: [:index, :show]
end
```

### Generated routes
``` ruby
dashboard_assignment GET        /dashboards/:dashboard_id/assignments/:id(.:format) assignments#show
  dashboard_students GET        /dashboards/:dashboard_id/students(.:format)        students#index
   dashboard_student GET        /dashboards/:dashboard_id/students/:id(.:format)    students#show
          dashboards GET        /dashboards(.:format)                               dashboards#index
           dashboard GET        /dashboards/:id(.:format)                           dashboards#show
```

### Controller action
``` ruby
def show
  @student_group = StudentGroup.find_by(name: params[:id])
  @assignments = @student_group.assignments.order('created_at ASC')
end
```

The problem was in an miscommunication between my routes and my controller. I was asking for the :id parameter, but was
fetching the :name from the database. This became even MORE apparent when I worked on the nested controllers.

``` ruby
def show
  @student_group = StudentGroup.find_by name: params[:dashboard_id]
  @assignment = Assignment.find params[:id]
end
```

There had to be a way to change that parameter to something better. Did the framework betray me?! Off to the docs

# The Docs

So many options for routes! Each one deserves its own post.  The docs are incredibly great, until they're not. I searched
and searched. and found NOTHING! Time to go code diving :-)

# How to Spelunk

My preferred method of diving through code is using Rubymine's Command + B feature. I knew I needed to pass some param to
resources in my routes file so I started there.

``` ruby
 def resources(*resources, &block)
  options = resources.extract_options!.dup
  ...
  resource_scope(:resources, Resource.new(resources.pop, options)) do
    ...
  end
  ...
end
```

Here I noticed we were creating a new Instance of the resource class with the options we passed in.

``` ruby
class Resource #:nodoc:
  attr_reader :controller, :path, :options, :param

  def initialize(entities, options = {})
    @name       = entities.to_s
    @path       = (options[:path] || @name).to_s
    @controller = (options[:controller] || @name).to_s
    @as         = options[:as]
    @param      = (options[:param] || :id).to_sym
    @options    = options
  end
  ...
end
```

LOOK AT THAT! You can pass in :param to the the resource declaration!


# The solution

Back in our routes file, we add the param option to the resource.

``` ruby
resources :dashboards, only: [:show, :index], param: :name do
  resources :assignments, only: [:show]
  resources :students, only: [:index, :show]
end
```

This generates these routes:

``` ruby
dashboard_assignment GET        /dashboards/:dashboard_name/assignments/:id(.:format) assignments#show
  dashboard_students GET        /dashboards/:dashboard_name/students(.:format)        students#index
   dashboard_student GET        /dashboards/:dashboard_name/students/:id(.:format)    students#show
          dashboards GET        /dashboards(.:format)                                 dashboards#index
           dashboard GET        /dashboards/:name(.:format)                           dashboards#show
```

And in our controllers:

``` ruby
def show
  @student_group = StudentGroup.find_by name: params[:dashboard_name]
  @assignment = Assignment.find params[:id]
end
```

So much better! I hope you found this useful. It was definitely fun to figure out.
