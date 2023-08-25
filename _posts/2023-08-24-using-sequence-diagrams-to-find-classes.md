---
layout: post
title: Using Sequence diagrams to find missing classes
author: Jorge Garcia
css: blog
date_published: "2023-10-06"
last_edited: "2023-10-06"
---

This is part of lessons learned from [__Practical Object Oriented Ruby Design__](https://www.poodr.com/) by Sandi Metz.

On the 4th chapter, she suggests the usage of sequence diagrams to find missing classes. There are a couple hints
that help us to discover classes, these are:
- A class that's doing too much outside its main responsibility
  - This can be found using the interrogation method. You rephrase each method as questions,
    the ones ones that don't make sense on the context of such class must belong elsewhere.
- The class Interacts with other classes asking for "what" instead of telling "how"
  - This means that a class should ask another class to how behave instead of asking the "what"
    and defining a business logic by its own.

This is a suggested action to do before coding, as it's cheaper than writing code and
having to rollback changes if needed.

### Context

We wanted to create a Data Pipeline (here it's called `DataFeed`) from resources of an
external REST-API provider. Alongside the next things:

- The `Provider` is a 3rd party app that my company integrates with.
- The external resources (called `trips`) would be sent to a Kafka Topic
- We request data frequently (every 20 minutes)
- Trips can be in progress, if they are, the next time we request data we need the timerange
  to include the in progress ones, so they're updated. The way we indentify ongoing trips
  from non-ongoing ones, is because the latest has a value in the `ends_at` that's not nil.

The **main** problem of this DataFeed, is to keep track of the oldest ongoing trip, as the next
enqueued job will start from that date.

Assuming the data feed runs for the first time, we have 2 posibilities that can occur, and the
time range will be:
```ruby
starts_at = Time.now - 1.day
ends_at = Time.now
``````

1. No `ongoing / in progress` trips; we will save the `ends_at` and the next time the
data feed runs, we will start from here.
2. At least 1 ongoing trip. We will search the oldest `starts_at` of the ongoing trips and
the next time the data feed run, we will start from here.

And repeat ^

**We already had something in place to keep track of this cursor:**

The trips belongs to a record called `GpsDevice`. And we used a `jsonb` column on these records to keep
track of the different cursors, not only trips but also other data feeds.

However, this wasn't `that` practical, each time we need to update the cursor,
we also need to `lock!` the device to prevent another data feed [overriding the content](https://stackoverflow.com/questions/43929857/append-only-jsonb-column-in-rails-postgres).

Code ex:
```ruby
device = GpsDevice.find(id)

device.api_metadata
{
  trips_next_pointer: "8-09-2023 13:30",
  sensor_data_next_pointer: "8-09-2023 12:55"
}

# Both data feeds can run at the same time so on each job we need to lock! the record
# to prevent overriding the jsonb column

device.with_lock! do |device|
  # find next pointer...
  device.api_metadata["trips_next_pointer"] = time

  device.save
end
```

Based on the above, that solution wasn't that practical. So I knew something wass odd.

The first sequence diagram was born following the existing implementation.

#### First Diagram

![diagram](/assets/images/blog/using_sequence_diagrams_to_find_classes/first_diagram.png)

Additional context:
- `DataFeeds::ProviderTrips` is the class that orchestrate requesting data, save the cursor and
  push the data to Kafka.
- `Provider::Client` is a wrapper around the provider API.
- `GpsDevice` is a model.
-  `ProviderTripProducer` Kafka producer.

This isn't that bad, but it has the next problem:
- `resolve_cursor` method seems like a simple one, but in reality it is not. It needs to
  iterate through the collection of trips and find the `starts_at` value of the oldest ongoing trip.

Let's try to imagine the private methods on the `DataFeed` we need to execute:
- get_trips (to the client)
- resolve_cursor!
  - iterate_trips
  - is_trip_finished?
  - is_trip_oldest_than_the_oldest_trip? (replace is yes)
- save_cursor
- push_trips_to_kafka

In order to determine if a method doesn't violate the Single Responsibility Principle, I like
to use the [interrogation method](https://medium.com/continuousdelivery/single-responsibility-principle-a67dde08f655).

OK `DataFeeds::ProviderTrip` answer me:
- what's your `get_trips?` ✅
- what's your `resolve_cursor!` ✅
- what's your `iterate_trips` 😕
- what's your `is_trip_finished?` 🫤
- what's your `is_trip_oldest_than_the_oldest_trip?` 🙈
- what's your `save_cursor` 🙈
- what's your `push_trips_to_kafka` ✅

It seems, the methods to resolve the cursor may not belong here?

*And dont get me started about the collection of trips having to be an instance and each one an instance as well*

Ok, let's do a second try:

#### Second Diagram

![diagram](/assets/images/blog/using_sequence_diagrams_to_find_classes/second_diagram.png)

This is much better!

However, the problem of `locking` the record persists.. Each `GpsDevice` is expected to be
updated multiple times per day + other DataFeeds using it. So if we can skip the `locking` would be ideal.
Also, this can be debatible, but I do think that storing the `cursor` on the `GpsDevice`
can be a violation of the SRP. A counter-argument can be that if we use the same object,
is 1 less object we need to query and I see that as a valid point.

And so the third diagram was born.

#### Third and Final Diagram
![diagram](/assets/images/blog/using_sequence_diagrams_to_find_classes/third_diagram.png)

With this approach, the cursor logic is consolidated in a single place. The `DataFeedCursor`
model. We're not violating the SRP anymore and the resolution of the cursor can be extended
as needed without impacting other classes.

### Summary
Take advantage of the sequence diagrams and use it as needed as they're a powerful yet
cheap way to validate our solutions, and found new classes.
