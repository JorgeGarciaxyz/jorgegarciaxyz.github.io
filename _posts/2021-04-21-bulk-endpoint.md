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

## Solution 1
------

```ruby
page_permissions_params.each do |page_permission|
  PagePermission.create(
    user_id: page_permission.user_id, page_id: page_permission.page_id
  )
end
```

I write the next simple test and ran it multiple times:

```ruby
# Note: this code sucks, I only wanted to provide context, not an example of a good performance test

def setup
  @users = []
  @pages = []

  5.times do
    @users << User.create
    @pages << Page.create
    PagePermission.create(page: @pages.first, user: @users.last)
  end

  @params = params
end

def params
  result = []
  @users.each do |user|
    @pages.each do |page|
      result << { page_id: page.id, user_id: user.id }
    end
  end
  result
end

test "bulk benchmark" do
  puts Benchmark.measure {
    post path, params: { page_permissions: @params }
  }
end

test "bulk memory" do
  Benchmark.memory do |x|
    x.report("endpoint") {
      post path, params: { page_permissions: @params }
    }
  end
end
```
note: I couldn't run rails performance tests, I found a weird issue on my test app. I
know this is probably not the best way to test this.

Results:
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

## Solution 2
------

Use [ActiveRecord-Import gem](https://github.com/zdennis/activerecord-import).

This gem execute a single insert statement.

```ruby
PagePermission.import(page_permissions_params, validate: true)
```

Same test as the solution 1, we have much better results

```
time: .02s
memory: 1.20mb
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

**What we want to do on the next solution?**
- Notify the client which records can not be created due to other circumstances rather than
  the record already existing
- If possible, not including existing record's attributes into the `insert` statement
