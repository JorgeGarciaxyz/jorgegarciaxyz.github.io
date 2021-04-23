---
layout: post
title: Bulk endpoint accepting existing records
author: Jorge Garcia
css: blog
---

In this article I'm gonna cover an interesting feature I developed a few months ago.

This feature was developed in the thrillshare.com application using Rails.

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
```

### The feature

*Given a list of users, I want to select N users and create permissions for N pages. If
the user already have access to any of these pages, we will not create new ones or
delete the existing ones, just ignore them.*

### Goals

- Few iterations as possible
- Few transactions as possible
- Simplicity, few complex operations as possible
