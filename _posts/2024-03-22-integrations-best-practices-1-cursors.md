---
layout: post
title: "Integrations Engineering Best Practices #1 Cursors"
title_shortened: "#1 Cursors"
author: Jorge Garcia
css: blog
date_published: "2024-03-22"
last_edited: "2024-03-22"
tags: integrations
---

This is an implementation of the [Cursor](https://glossary.airbyte.com/term/cursor/) concept in Data Engineering.

Cursors need to be used with [Incremental Synchronization](https://glossary.airbyte.com/term/incremental-synchronization/).

A `Cursor` is an object that keeps track of the last parameters used to request data from an API,
used when you need to request data frequently.

Beyond the documentation I shared above, the cursor can also help in other ways.
Here, I'll present a common use case to avoid data loss.

## Problem

You need to pull data from an API to create `something` in your app. (Creating some form of
[Data Pipeline](https://www.ibm.com/topics/data-pipeline#:~:text=A%20data%20pipeline%20is%20a,usually%20undergoes%20some%20data%20processing.)).

I'm going to present an example of how Cursors can be used with these pipelines.

A simple way to create this pipeline is by executing cron jobs, with (at least in the Rails world) a background job framework.

Example: let's say we work on an app that pulls online messages from different platforms.
We want to create a `Slack` integration to sync messages.

```yml
# Sidekiq scheduler
SyncMessagesJob:
  every: "5 minutes"
```

```ruby
class Message < ApplicationRecord
end

class SlackIntegration < ApplicationRecord
end

class SyncMessagesJob
  def perform
    SlackIntegration.all.each do |slack_integration|
      to = Time.zone.now
      from = starts_at - 5.minutes

      # The `from` and `to` is a time range to request messages created within this time range.
      messages = slack_integration.get_messages(from: from, to: to)

      Message.upsert(messages)
    end
  end
end
```

Lets ignore the next things:
- Splitting the job into many concurrent jobs make it more performant.

This is good enough, but after an analysis, you **found that `some` data is missing.** And you
probably require a backfilling process. But why does this happen?

- You schedule the cron to run every 5 minutes, but this doesn't guarantee that each job will run precisely every 5 minutes.
- If the queue is full, or if the previous job is still running, the jobs may take a bit longer.
- Even if we split the jobs in parallel, the problem is still present.

Let's picture this with timestamps:

1. The job starts running exactly at `10:05:00`
2. X Integration runs the job with the values `from = 10:00:01` and `to = 10:05:01`
2. Between the next run (from 10:05 -> 10:10) another job packed the queue, now is full.
3. The next job is scheduled at `10:10`, but it take some time to finally be processed.
4. X Integration runs the job, with the values `from = 10:05:45` and `to = 10:10:45`
5. **You can see there's a time range we're not covering, from `10:05:02` -> `10:05:44`**

Let's discuss a few ideas:

- You can technically solve this problem if you sync once per day or every week.
  But the throughput to get the data of this time range can be considerable. You will not always be able to do this.
- A backfilling job that runs a few times per day or so can also solve this problem,
  partially. It can be pretty annoying to wait a few hours for the complete data to be available.

## Solution

A `Cursor` is a simple object that at minimun, it saves the next attributes:
- `next_value` saves the value that'd be used next time the pipeline runs.
  - This ensures that no matter the time the pipeline runs, we will always cover every time range.
- (optional) an association to a certain object
  - To re-use this you can use a polymoprhic association (Rails)

The `Cursor` object is used to keep track of the next params to use and updates this
value when the job finishes.

## Usage

Models:

```ruby
class Cursor < ApplicationRecord
  belongs_to :slack_integration # or polymorphic if needed
end

class SlackIntegration < ApplicationRecord
  has_one :cursor # or has_many if you have multiple type of objects you need to integrate with
end
```

Job:

```ruby
class SyncMessagesJob
  def perform
    SlackIntegration.eager_load(:cursor).all.each do |slack_integration|
      from = integration.cursor.next_value # or use find_or_initialize_by if doesn't exist
      to = Time.zone.now

      messages = integration.get_messages(from: from, to: to)

      Messages.upsert(messages)

      integration.cursor.update(next_value: to)
      # NOTE: the cursor is updated after the upsert. If by some reason, the cursor can't be
      #       updated, we will not loose data, as at worst the upsert will be executed again.
      #       Otherwise (updating before the upsert) we can lose data if the upsert fails.
    end
  end
end
```

Let's picture the previous example:

1. The job starts running exactly at `10:05:00`
2. X Integration runs the job, the `cursor.next_value` is `10:00:02` we use this as `from`.
   Now is the same as the previous example, `to = 10:05:01`
3. Between the next run (from 10:05 -> 10:10) another job packed the queue, now is full.
4. The next job is scheduled at `10:10`, but it take some time to finally be processed.
5. X Integration runs the job, the `cursor.next_value` is `10:05:01` we use this as `from`
   and `to = 10:10:45`
6. This time (because we save the previous value as `next_value`) we don't loose data, as
   the time range is always covered.

Cons:
- If by some reason the Cursor can't be updated, the next time this runs the time range
  will be bigger.
- Now on each job we need to make one more `update` statement, causing more stress to the db.
  - Although as long as the model is simple it shouldn't be a big problem vs downsides of the
    previous approach.
