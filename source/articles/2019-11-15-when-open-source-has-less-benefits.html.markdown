---
layout: post
title: When open source has less benefits
date: 2019-11-15 22:42 UTC
tags: [ruby]
---

Here at [Shopify](https://www.shopify.com/), I work as a Production Engineer on the Jobs team. Our mission is to maintain and improve the background job infrastructure for Shopify Core, which happens to be one of the largest Ruby on Rails applications in the World.

READMORE

Let's start with some context about Background Jobs, especially in the context of ruby applications.
When talking about jobs, we refer to those units of work that are important for the application to function but would take too long to be processed within the lifetime of a web request. The usual way of dealing with those units is to offload them to the background, aka "background jobs", and for a different process to pick those jobs and execute them.

In Ruby land, there are many popular options for dealing with jobs. The most commonly used one is [Sidekiq](https://sidekiq.org/) and we have others like [Resque](https://github.com/resque/resque), [Delayed job](https://github.com/tobi/delayed_job), [Sucker punch](https://github.com/brandonhilkert/sucker_punch)...

This post is not going to focus on how those libraries work or what are their differences, but rather why at Shopify, we decided to build our internal library for dealing with jobs.

To provide a little more context, at Shopify, in the beginning, we were using Delayed job, which was created by Tobias Lütke, current CEO of Shopify. The backend used to store jobs was [MySQL](https://www.mysql.com/), and at some point, MySQL wasn't able to keep with the amount of writes jobs were creating. Later [Github released Resque in November of 2009](https://github.blog/2009-11-03-introducing-resque/), which uses [Redis](https://redis.io/) as the backend to store jobs. At that time, it was the only option that could support the requirements of Shopify. Since then, we have added functionality on top of Resque with external gems like [resque-scheduler](https://github.com/resque/resque-scheduler) or [resque-pool](https://github.com/nevans/resque-pool).

Of course, the requirements of the application always change, and the library that you are using might not evolve the way that would satisfy your new needs.

Throughout the years of Shopify, the complexity of our monolith kept increasing as our needs kept evolving. As a result, we introduced sharding and new sophisticated job primitives. For many years, `resque` and the other gems helped to solve these problems, but those libraries could no longer provide all the functionality that Shopify needed. We had to regularly hack or patch the library so we could keep providing the functionality that was required.

You can imagine that for the maintainers of this system, this was not ideal. We were working with at least three `resque-*` gems and dozens of custom modules that provided extra functionality. This situation was the cause of frustration, production errors, and a complex enough system that it was difficult to grasp in one's head. Most critically, adding new features was laborious and error-prone.

At the beginning of this year, my team decided that we were going to aggregate all resque code and all the custom functionality that we have built over the years and place everything under one library.

After four months of work, we are currently running all jobs from the monolith with our library, and we have ported all previous patches into our library at the time this post is published.

This decision has allowed the team to work on major refactors to the internals of the job infrastructure and adapt to new requirements that appeared throughout the year.

Thanks to that decision, adding those new features was smooth, and the team confidence on the code base has increased substantially, more importantly, we are in a much better place to keep improving the job infrastructure.

Here at Shopify, we value open-source projects, and we contribute to them as much as we can. But even though that is the mentality at Shopify, there are occasions were owning all of the code has more benefits in the long term.

The conclusion that I wanted to leave here is that if you or your team find yourselves often working around library functionality, finding hacky ways to extend it or change the core functionality of those libraries, sometimes it's better to rethink the approach and question what you need from those libraries. If you need only 20% and end up hacking the rest of 80%, you might get benefits from owning all of the code.

If you have any thoughts or questions, please share them, and I will be happy to answer in the comments.



