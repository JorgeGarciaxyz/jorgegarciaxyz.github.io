---
layout: post
title: Integrations Engineering Best Practices
author: Jorge Garcia
css: blog
date_published: "2024-03-22"
last_edited: "2024-03-22"
---


Since joining Fleetio in September 2021, I've been part of the Integrations team, tasked
with connecting Fleetio with other platforms, primarily telematics and fuel cards systems.
Here's a link to learn more about [Fleetio's integrations](https://www.fleetio.com/solutions/integrations).

To encapsulate my role, we pull data from various platforms and use it to create records in Fleetio.
Or utilize Fleetio data to generate records in other platforms. Part of my job is to maintain
some form of Data Pipelines.

While the title "Software Integrations Engineer" may hold different meanings across various companies,
the description I provided best encapsulates my responsibilities.

After few years in this role, gathering insights from both work and personal experiences,
I've identified certain practices that could prove beneficial in integrations.
I've decided to compile them here due to the scarcity of content on this subject.

Given the broad spectrum that "Integrations" encompasses,I'll delineate various use cases
for each as there's no one-size-fits-all solution.

### Integrations Best Practices Posts

{% assign integrations_posts = site.posts | where_exp: "post", "post.tags contains 'integrations'" %}

{% for post in integrations_posts %}
  - [{{ post.title_shortened }}]({{post.url}})
{% endfor %}
