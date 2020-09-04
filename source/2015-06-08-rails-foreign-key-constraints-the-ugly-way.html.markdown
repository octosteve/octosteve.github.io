---
title: Rails Foreign Key Constraints - The Ugly Way
date: 2015-06-08 22:43 UTC
tags: rails, postgresql
---

# Render unto Caesar...
Let the database do it's job. For a while, Rails development focused on pushing logic into models that would be better suited being put in as a table constraint. Articles like [this one](http://shuber.io/porting-activerecord-validations-to-postgres) are great at outlining the proper approach. The problem with this is that it's not built into the Rails tools, leaving you writing ugly code in your migrations. Not very ruby-esque.

## Foreign Keys
Databases have a built in way for managing data integrity between 2 tables. [The Foreign Key constraint](http://en.wikipedia.org/wiki/Foreign_key). This ensures the value of a foreign key exists in the referenced table.

Since Rails was created, it relied on SQL, and foreign keys, but never had a way of confirming the `id` being referenced existed, shy of adding a model validation.

Exiting news! Rails 4.2 shipped with foreign key support. You can create migrations that delegate the responsibility directly to the database if your database supports it.

There are other articles that cover [some](https://robots.thoughtbot.com/referential-integrity-with-foreign-keys) of the [basics](https://richonrails.com/articles/foreign-keys-in-rails-4-2). I wanted to cover something I ran into recently.

## Learn stuff
We're going to model an app that lets you teach and sign up for courses.

```
User -> Can be both a teacher and a student
Course -> Belongs to a user by a teacher_id
Enrollment -> Links a user by a student_id, and a course
```

### Some rails

``` shell
rails new learn_stuff -d postgresql && cd learn_stuff
rails g model user name
rails g model course topic teacher:belongs_to
rails g model enrollment student:belongs_to course:belongs_to
rake db:create
```

This sets us up, but when we run `rake db:migrate` fireworks start.

```
PG::UndefinedTable: ERROR:  relation "teachers" does not exist
```

This makes sense. We don't have a `teachers` table. We need to pass the course table some more options.

Here's our migration as it stands.

``` ruby
class CreateCourses < ActiveRecord::Migration
  def change
    create_table :courses do |t|
      t.string :topic
      t.belongs_to :teacher, index: true, foreign_key: true

      t.timestamps null: false
    end
  end
end
```

The Rails [source](https://github.com/rails/rails/blob/7785417984f61a9d5e00416c13b89dce2ee02daf/activerecord/lib/active_record/connection_adapters/abstract/schema_statements.rb#L658) shows that we can't switch the name of table. What would be great is to be able to say something like: "OK RAILS, I'M GOING TO GIVE YOU A COLUMN NAME BUT IT'S NOT THE TABLE NAME FOR THE CONSTRAINT. I'LL GIVE YOU MORE INFO LATER. KTHXBAI"

We have to first create the column, THEN add the constraint.

``` ruby
class CreateCourses < ActiveRecord::Migration
  def change
    create_table :courses do |t|
      t.string :topic
      t.belongs_to :teacher, index: true

      t.timestamps null: false
    end
    add_foreign_key :courses, :users, column: :teacher_id
  end
end
```

The rest of the source code, including the associations for all of this stuff can be found [here](https://github.com/octosteve/learn_stuff)

### Meh

I wish this was a bit smarter, off to work on a pull request.
