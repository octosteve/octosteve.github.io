---
title: Filtering through subqueries in ActiveRecord
date: 2019-12-12 01:49 UTC
tags: queries, rails, sql
---
I had a requirement this week to implement some filtering logic. This post is a collection of my learnings. The domain has been changed a bit the concepts are the same.

## Course Tracker
Code can be found [Here](https://github.com/octosteve/course_tracker). The domain is pretty straight forward. For our purposes, let's focus on the Courses.

A Course has many `ratings`. One query we're interested in is finding a course with a minimum average rating.

In SQL, fetching courses with a rating of 2 or higher would look something like:

```sql
SELECT *
FROM courses
INNER JOIN ratings
ON ratings.course_id = courses.id
GROUP BY courses.id
HAVING AVG(ratings.rating) > 2
```

ActiveRecord makes this super easy:

```ruby
Course.joins(:ratings).group(:id).having("AVG(ratings.rating) > 2")
# => #<ActiveRecord::Relation [#<Course id: 1, name: "Cooking", created_at: "2019-12-12 00:49:38", updated_at: "2019-12-12 00:49:38">, #<Course id: 2, name: "Learn to Code", created_at: "2019-12-12 00:49:38", updated_at: "2019-12-12 00:49:38">]>
```

RUBY I LOVE YOU ❤️ ❤️ ❤️. This returns 2 records. Both with an average rating over 2.

Our next filtering task was to find courses matching a list of tags. This one is tricky since we want to only return courses that match ALL the tags.

Our [seed data](https://github.com/octosteve/course_tracker/blob/master/db/seeds.rb#L25) shows that we have a few courses that are tagged as "Fun", but only one that is also tagged as "Food"

What would this look like in SQL if we were looking for "Fun" and "Food"?

```sql
SELECT *
FROM courses
INNER JOIN taggings
ON taggings.course_id = courses.id
INNER JOIN tags
ON taggings.tag_id = tags.id
WHERE tags.name IN ("Fun", "Food")
GROUP BY courses.id
HAVING COUNT(*) = 2
```

This one is funky. Let's walk through the thinking.
If a tag doesn't match at all, it won't produce a row since we're using an `INNER JOIN`.
If only 1 of our tags match, then we'll get a row that describes the course and JUST that tag.

However (this is the cool bit), if both tags match, we'll get 2(!) with the same course, but with the tag that matched.

Take a look at the result without any of the `GROUP BY` and `HAVING` business.

```sql
SELECT courses.name, tags.name
FROM courses
INNER JOIN taggings
ON taggings.course_id = courses.id
INNER JOIN tags
ON taggings.tag_id = tags.id
WHERE tags.name IN ("Fun", "Food")
--name        name      
  ----------  ----------
--Cooking     Food      
--Cooking     Fun       
--Learn to C  Fun    
```
We want the one that matches the count of the tags we passed in.

After the filtering we're left with just the one Course.

```sql
SELECT courses.name, tags.name
FROM courses
INNER JOIN taggings
ON taggings.course_id = courses.id
INNER JOIN tags
ON taggings.tag_id = tags.id
WHERE tags.name IN ("Fun", "Food")
GROUP BY courses.id
HAVING COUNT(*) = 2;
--name        name      
------------  ----------
--Cooking     Food 
```

In ActiveRecord, this looks something like this:

```ruby
tags = ["Food", "Fun"]
Course.joins(:tags).where(tags: {name: tags}).group(:id).having("COUNT(*) = ?", tags.count)
=> #<ActiveRecord::Relation [#<Course id: 1, name: "Cooking", created_at: "2019-12-12 00:49:38", updated_at: "2019-12-12 00:49:38">]>
```

## Filters on Filters
This works, but what we're trying to do is refine results like a sieve. In SQL, you'd use subqueries for that.

Say we wanted to combine our 2 examples, finding courses with a rating of 2 and above, AND with the tags of "Food" and "Fun". That's a lot of constraints! We could so something like this in SQL:

```sql
SELECT *
FROM courses
INNER JOIN taggings
ON taggings.course_id = courses.id
INNER JOIN tags
ON taggings.tag_id = tags.id
WHERE tags.name IN ("Fun", "Food")
AND courses.id IN (SELECT courses.id
                   FROM courses
                   INNER JOIN ratings
                   ON ratings.course_id = courses.id
                   -- WHERE (further constraints go here...)
                   GROUP BY courses.id
                   HAVING AVG(ratings.rating) > 2)
GROUP BY courses.id
HAVING COUNT(*) = 2;
```
SUBQUERIES. We'd continue to refine the queries with more subqueries if needed.

What does this look like in ActiveRecord? SO EASY.

```ruby
rating = 2
tags = ["Food", "Fun"]
ratings_query = Course.joins(:ratings).group(:id).having("avg(ratings.rating) > ?", rating)
Course.where(id: ratings_query.all).joins(:tags).where(tags: {name: tags}).group("id").having("COUNT(*) = ?", tags.count)
=> #<ActiveRecord::Relation [#<Course id: 1, name: "Cooking", created_at: "2019-12-12 00:49:38", updated_at: "2019-12-12 00:49:38">]>
```

It works! Not only that, take a look at what calling `to_sql` returns from our second query.

```sql
SELECT "courses".*
FROM "courses"
INNER JOIN "taggings"
ON "taggings"."course_id" = "courses"."id"
INNER JOIN "tags" ON "tags"."id" = "taggings"."tag_id"
WHERE "courses"."id" IN (
                    SELECT "courses"."id"
                    FROM "courses"
                    INNER JOIN "ratings"
                    ON "ratings"."course_id" = "courses"."id"
                    GROUP BY "courses"."id"
                    HAVING (avg(ratings.rating) > 2)
                    )
AND "tags"."name" IN ('Food', 'Fun')
GROUP BY "courses"."id"
HAVING (COUNT(*) = 2)
```

Thanks ActiveRecord!!

## Putting it all together.
Let's define an interface for these queries.

Calling `CourseQueryBuilder.new.run(tag: ["Food", "Fun"], min_rating: 2)` should get us the same record

First thing we do is always make _all_ our queries take a `where` subquery and wrap them in methods. Those methods should receive the previous query.

```ruby
class CourseQueryBuilder
  def run(params)
   # More to come 
  end
  def by_tag_list(query, tags)
    Course.where(id: query.all).joins(:tags).where(tags: {name: tags}).group("id").having("COUNT(*) = ?", tags.count)
  end
  def by_min_rating(query, rating)
    Course.where(id: query.all).joins(:ratings).group(:id).having("avg(ratings.rating) > ?", rating)
  end
end
```

Next let's define `#run`. In this method we're going to iterate over all of the facets (keys) and constraints (values) provided. If we find something we can filter, we call the corresponding method, if we don't we return the last returned query. Sounds like a job for `Enum.reduce`!

```ruby
class CourseQueryBuilder
  def run(params)
    params.reduce(Course) do |query, (facet, constraint)|
      case facet
      when :tag
        by_tag_list(query, constraint)
      when :min_rating
        by_min_rating(query, constraint)
      else
        query
      end
    end
  end

  def by_tag_list(query, tags)
    Course.where(id: query.all).joins(:tags).where(tags: {name: tags}).group("id").having("COUNT(*) = ?", tags.count)
  end

  def by_min_rating(query, rating)
    Course.where(id: query.all).joins(:ratings).group(:id).having("avg(ratings.rating) > ?", rating)
  end
end
```

We pass in `Course` as the first query since we want to start with all records. Each reduction we reduce the amount of records by applying more queries.

Here's our rating query returning our 2 Courses:

```ruby
CourseQueryBuilder.new.run(min_rating: 2)
#=> #<ActiveRecord::Relation [#<Course id: 1, name: "Cooking", created_at: "2019-12-12 00:49:38", updated_at: "2019-12-12 00:49:38">, #<Course id: 2, name: "Learn to Code", created_at: "2019-12-12 00:49:38", updated_at: "2019-12-12 00:49:38">]>
```

And here's the combined result:

```ruby
CourseQueryBuilder.new.run(min_rating: 2, tag: ["Food", "Fun"])
#=> #<ActiveRecord::Relation [#<Course id: 1, name: "Cooking", created_at: "2019-12-12 00:49:38", updated_at: "2019-12-12 00:49:38">]>
```

## Summary
Subqueries give you a ton of power by allowing you to write simple composable code that's easy to maintain, and understand.

I hope this was useful!
