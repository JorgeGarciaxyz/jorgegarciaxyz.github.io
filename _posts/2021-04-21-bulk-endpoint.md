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

On Thrillshare we have the model `PagePermission`, the user can have many permissions,
and each permission refers to a single page.


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
the user already has access to any of these pages, we will not create new ones or
delete the existing ones, just ignore them.*

There is a tricky part of this feature, we don't receive the `ids` of the existing
permissions, on the app, you only select a `user_id` alongside a `page_id`.

### Goals

- Few iterations, transactions, and queries as possible
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

Let's go first with the simplest solution, iterating the array and create one
PagePermission on each iteration.


Note: The code here is not 100% accurate with the code in production,
but it is kinda similar. Most of this stuff is for test purposes.

I'm gonna start with a simple solution and iterate the idea.

## 1st Solution
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

This solution works just fine, is  [good enough software](https://en.wikipedia.org/wiki/Principle_of_good_enough)
but as I am ~~sometimes~~ obsessive with the performance, I wanted to go further into this
problem.

**The problem with this solution**
- We have the n+1 insert problem
- As the payload size grows, the performance gets worse

Let's improve this

## 2nd Solution
------

Use [ActiveRecord-Import gem](https://github.com/zdennis/activerecord-import).

This gem executes a single insert statement.

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
  - As Thrillshare is widely used, there is a possibility that at the time a user sends
    a request to this endpoint, a Page or a User is deleted, thus some records can not be created.

Alongside the mentioned problems, I’m omitting an important one. Why we’re trying to
insert records that we know for sure that shouldn’t be created? I'm talking about the
existing PagePermissions. I mean, is for simplicity but if we can avoid it, I’ll take it.

## 3rd Solution
------

Goals:

- Notify the client which records can not be created due to other circumstances rather than
  the record already existing
    - activerecord-import supports this using `failed_instances` on the returned object
      of the import method [link](https://github.com/zdennis/activerecord-import#return-info)
- If possible, not including existing record's attributes into the `insert` statement

A simple solution can be the next one:

```ruby
page_permissions_params_without_existing_records = page_permissions_params.filter_map do |page_permission|
  page_permission unless PagePermission.find_by(**page_permission)
end

bulk_creation = PagePermission.import(
  page_permissions_params_without_existing_records, validate: true
)

handle_errors(bulk_creation.failed_instances) if bulk_creation.failed_instances.present?
```

Performance results:
```
time: .042s
memory: 652k
```

It's similar to the 2nd solution but we have a big problem here; we're executing N+1 Select
statements. So the performance would just get worse if the payload grows.

## 4th Solution
------

Goals:
- Exclude the existing records from the insert statement without having an N+1 `SELECT` problem.

**Idea 1:**
Execute a single `SELECT` statement to find the existing records, then remove the existing
records from the `INSERT` statement doing a hash diff.

**Idea 2**
Use relational algebra to find the non-existing records and use the result to execute
the `INSERT` statement.

I didn't like idea #1 `that much` specifically the hash diff. If I have the option
to choose between iterating through hashes or using SQL, I prefer SQL. It may be worth it
to explore this solution but for now, I did not take it. I went with the #2.

**Exploring the Idea 2**

We want to get the non-existing records from the params, the existing ones are stored in
a table. So what we're trying to do is basically a `LEFT JOIN`! But how we can do it?

There is no table for the non-existing records... but we can use a temporary table.

Let's do it.

Please note that temporary tables must be wrapped inside a transaction.

```ruby
bulk_creation = nil

ActiveRecord::Base.transaction do
  create_temporary_page_permission_table
  create_temporary_indexes
  create_temporary_page_permissions

  bulk_creation = PagePermission.import(new_permissions, validate: true)
end

handle_errors(bulk_creation.failed_instances) if bulk_creation.failed_instances.present?

def create_temporary_page_permission_table
  ActiveRecord::Base.connection.execute <<~SQL
    CREATE TEMP TABLE temporary_page_permissions (
      user_id BIG INTEGER NOT NULL,
      page_id BIG INTEGER NOT NULL
    )
  SQL
end

def create_temporary_indexes
  ActiveRecord::Base.connection.execute <<~SQL
    CREATE INDEX index_temporary_page_permissions_on_user_id ON
      temporary_page_permissions (user_id);
    CREATE INDEX index_temporary_page_permissions_on_page_id ON
      temporary_page_permissions (page_id);
  SQL
end

def create_temporary_page_permissions
  values = transform_params_to_values_string

  ActiveRecord::Base.connection.execute <<~SQL
    INSERT INTO temporary_page_permissions (user_id, page_id) VALUES #{values}
  SQL
end

def transform_params_to_values_string
  values = "".dup

  page_permissions_params.each do |page_permission|
    values.concat(
      "(#{page_permission[:user_id]}, #{page_permission[:page_id]}), "
    )
  end
  values.chomp(", ")
end

def new_permissions
  sql = <<~SQL
    SELECT temporary_page_permissions.user_id, temporary_page_permissions.page_id
    FROM temporary_page_permissions
    LEFT JOIN page_permissions ON temporary_page_permissions.user_id = page_permissions.user_id
      AND temporary_page_permissions.page_id = page_permissions.page_id
    WHERE page_permissions.user_id IS NULL AND page_permissions.page_id IS NULL
  SQL

  ActiveRecord::Base.connection.execute(sql).to_a
end
```

Performance results:
```
time: .033s
memory: 509k
```

With this solution, we're executing: 2 `INSERT` statements, 1 `SELECT` statement so we
successfully get rid of the N+1 problems. There can be a bottleneck tho, the method
`transform_params_to_values_string` can cause some problems if the payload is too big.
Not too much concern for now at least, but is something we need to consider.

## Results and conclusion
------

| Solution #  | Time        | Memory      |
| ----------- | ----------- | ----------- |
| 1           | 0.080s      | 2.74mb      |
| 2           | 0.036s      | 509k        |
| 3           | 0.042s      | 652k        |
| 4           | 0.033s      | 509k        |

The #4 has the best results and fulfill all goals
- Few iterations, transactions, and queries as possible
- Simplicity (kinda if you're familiar with SQL)
- Efficient
- Existing records are not included in the `INSERT` statement to create the permissions
- The user is notified which records can not be created (omitting the existing ones)

I'm really happy with this solution, I made it almost a year ago and is working fine.
We still didn't have to make changes there so the time would tell if this was the best solution or not.

This is one of my favorites problems because I needed to think differently, my solution
was not too intuitive and I explore multiple approaches and end up with a creative one.
