---
layout: post
title: Bulk endpoint accepting existing records
author: Jorge Garcia
css: blog
---

In this article I'm gonna cover an interesting feature I developed a few months ago.

This feature was developed in the thrillshare.com application using Rails on an API endpoint.

*Note: I simplify some of the models so they're not 100% accurate from what we have*

### Context

On Thrillshare we have the model `PagePermission`, the user can have many permisisons
and each permission reffer to a single page.


```ruby
PagePermission.new(user_id: user_id, page_id: page_id)

# User model

has_many :page_permissions

# PagePermission model

belongs_to :user
belongs_to :page

validates :page, presence: true, uniqueness: { scope: :user_id }
validates :user, presence: true
```

### The feature

*Given a list of users, I want to select N users and create permissions for N pages. If
the user already have access to any of these pages, we will not create new ones or
delete the existing ones, just ignore them.*

There is a tricky part on this feature, we don't receive the `ids` of the existing
permissions, on the app, you only select a `user_id` alongside a `page_id`.

### Goals

- Few iterations, transactions and queries as possible
- Simplicity
- Efficient


#### Payload

```json
{
  "page_permissions": [
    { "user_id": 1, "page_id": 1 },
    { "user_id": 2, "page_id": 1 },
    { "user_id": 2, "page_id": 2 },
    { "user_id": 3, "page_id": 2 }
  ]
}
```

## How to solve this problem?
------

Lets go first with the simplest solution, iterating the array and create one `PagePermission`
on each iteration.

Note: The code here is not 100% accurate with the code in production, but its kinda similar.
Most of this stuff is for test purposes.

## 1st iteration of the solution
------

```ruby
page_permissions_params.each do |page_permission|
  PagePermission.create(**page_permission)
end
```

I couldn't run rails performance tests, I found a weird issue on my test app. Given time
constraints I write the next simple test but I know is probably not the best way to
benchmark this problem.

```ruby
def setup
  @users = []
  @pages = []

  5.times do
    @users << User.create
    @pages << Page.create
    PagePermission.create(page: @pages.first, user: @users.last)
  end

  generate_params
end

def generate_params
  @params = []

  @users.each do |user|
    @pages.each do |page|
      @params << { page_id: page.id, user_id: user.id }
    end
  end
end

test "Benchmark time" do
  puts Benchmark.measure {
    post path, params: { page_permissions: @params }
  }
end

test "Benchmark memory" do
  Benchmark.memory do |x|
    x.report("endpoint") {
      post path, params: { page_permissions: @params }
    }
  end
end
```

Results after running multiple times (around 5) and calculating means:
```
time: .08s
memory: 2.74mb
```

This solution works just fine, is a [good enough software](https://en.wikipedia.org/wiki/Principle_of_good_enough)
but as I am ~~sometimes~~ obsesive with the performance, I wanted to go further into this
problem.

**The problem with this solution**
- We have the n+1 insert problem
- As the payload size grows, the performance gets worse

Lets improve this

## 2nd iteration
------

Use [ActiveRecord-Import gem](https://github.com/zdennis/activerecord-import).

This gem execute a single insert statement.

```ruby
PagePermission.import(page_permissions_params, validate: true)
```

Running the same performance test, we have better results:

```
time: .036s
memory: 509k
```

**The problem with this solution**

The endpoint allows partial creation of resources. Mixed between existing records and
invalid ones. We need to return on the request useful information about why some records
couldn't be created. Thus we need to know which are the records who:
- Failed for existing records
- Failed for other validations
  - As Thrillshare is widly used, there is a posibility that at the time a user sends a request
    to this endpoint, a Page or a User is deleted, thus some records can not be created.

Alongside the mentioned problems, I'm omiting an important one.
Why we're trying to insert records that we know for sure that shouldn't be created? Im talking
about the existing `PagePermissions`. I mean, is for simplicity but if we can avoid it, I'll take it.

## 3rd iteration
------

Goals:

- Notify the client which records can not be created due to other circumstances rather than
  the record already existing
- If possible, not including existing record's attributes into the `insert` statement

A simple solution can be the next one:

```ruby
page_permissions_params_without_existing_records = page_permissions_params.filter_map do |page_permission|
  page_permission unless PagePermission.find_by(**page_permission)
end

PagePermission.import(page_permissions_params_without_existing_records, validate: true)
```

Performance results:
```
time: .042s
memory: 652k
```

It's similar to the 2nd iteration but we have a big problem here; we're executing N+1 Select
statements. So the performance would just get worse if the payload grows.
