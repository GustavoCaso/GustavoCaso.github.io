+++
title = 'Migrating millions of Redis keys without downtime'
date = 2019-04-30
tags = ["ruby", "redis"]
+++

Last year in September I joined the Job Patterns team at [Shopify](https://www.shopify.com/).

The mission of the team is to provide a stable platform so that developers can write their background jobs to power one of the biggest e-commerce platforms in the world.



Since I joined, I have been gathering context around the various components that create the unique Shopify ecosystem.

To provide some context, Shopify is a [massive Ruby on Rails monolith](https://stackshare.io/shopify/e-commerce-at-scale-inside-shopifys-tech-stack) application and the background job architecture consists of [ActiveJob](https://edgeguides.rubyonrails.org/active_job_basics.html), [Resque](https://github.com/resque/resque), and [Redis](https://redis.io/). Besides the functionality that those libraries provide by default, we have created many additional modules that allow developers to define custom behaviour for their jobs: `Locking`, `LockQueue`, `Concurrency`, `Retry`, `Status`, and many more.

At Shopify, we have many Redis instances; Each instance stores information that belongs to different parts of the platform.

This post is going to focus on how we managed to migrate millions of keys from one of our Redis instances to another without downtime or incidents.

The module we'll be discussing here is the `Locking` module. Developers use this module to prevent multiple jobs of the same class with the same arguments to be executed by multiple processes at the same time. It provides the same functionality as [unique jobs](https://github.com/mhenrixon/sidekiq-unique-jobs) for Sidekiq.

Before enqueuing the job, it checks for the existence of the lock key. If the lock key does not exist, we acquire it until the job is done and finally release it. If the lock key does exist at the time of enqueueing it means that another job already exists, so we do not enqueue the new job.

## Improving performance

At the current growth rate of Shopify, we are looking into multiple ways to optimize the background jobs infrastructure for performance.

To reduce load from a single Redis for jobs queues, we plan on deploying more Redis instances so we can multiplex both the enqueue and dequeue operations.

A blocker for this idea is that we would need to know at all times where the unique locks are stored 🤔.

We decided on the solution to move lock keys from the Redis instance holding the queue information to a separate Redis instance. That way, we know at all times where the lock keys are stored unlike job queues that could span across multiple Redis instances in the future.


We process hundreds of thousands of jobs per minute, and those jobs are time-sensitive, so stopping the system, migrating the keys and deploying the changes is not a possible solution for us. We, therefore, had to perform the migration without a maintenance window or downtime.

How did we manage to achieve this?

We devised a 3-step plan that would allow us to do it. All steps required code changes in the application, so the full migration took roughly 2 weeks.

## Implementation

Let's introduce our `Locking` module. The following is going to be a simplified version of the one currently maintained at Shopify:

```ruby
class Locking
  AlreadyAcquireLockError = Class.new(StandardError)

  attr_reader :lock_key, :token

  def initialize(lock_key, token: SecureRandom.uuid)
    @lock_key = lock_key
    @token = token
    @have_lock = false
  end

  def have_lock?
    @have_lock
  end

  def acquire(duration)
    raise AlreadyAcquireLockError if have_lock?
    @have_lock = redis.set(key, token, ex: duration, nx: true)
  end

  def relase
    redis.del(key)
    @have_lock = false
  end

  def locked?
    redis.exists(key)
  end

  private

  def redis
    Resque.redis
  end
end
```

In the following steps, we will refer to the Redis instance holding the queue information as the `jobs` Redis (the source of the migration), and the Redis instance holding the locks information as the `locks` Redis (the destination of the migration).


**First Step:**

We modify the `locked?` method to check on the locks Redis and then on the resque Redis. With this change, the functionality stays the same, but we introduce the locks Redis as a new dependency.

```ruby
def locked?
  redises.each do |redis|
    break(true) if redis.exists(key)
  end
  false
end

private

def redis
  Resque.redis
end

def redises
  [Lock.redis, redis]
end
```

**Second Step:**

We are going to start `acquiring` the lock key on the `locks` Redis. The `release` method tries to release the lock from the `locks` Redis instance first, and if not successful, it will try releasing the lock from the Resque Redis instance. The `locked?` method stays the same as in the first step.

```ruby
def acquire
  raise AlreadyAcquireLockError if have_lock?
  @have_lock = lock_redis.set(key, token, ex: duration, nx: true)
end

def release
  redises.each do |redis|
    # redis returns the number of keys deleted
    if redis.del(key) > 0
      @have_lock = false
      break
    end
  end
end

private

def redis
  Resque.redis
end

def lock_redis
  Lock.redis
end

def redises
  [lock_redis, redis]
end
```

__Note:__ After deploying this change, we monitored the platform for a couple of days to make sure everything was working as expected (meaning, lock keys were being acquired and released without any issue).

**Last Step:**

We change all the code to make sure that the only Redis instance involved with the `Locking` module is the `locks` Redis. All acquiring, releasing and checking actions of the keys have now been migrated over.


```ruby
private

def redis
  Lock.redis
end
```

With these steps, we were able to migrate the lock keys successfully without impacting the platform 🎉 🎉.

Before starting the migration we asked ourselves questions like: Would the `locks` Redis be able to handle the load? Is the `locks` Redis a single point of failure?

The changes weren’t as straightforward as described above. There were other components involved, many tests to modify and some infrastructure changes to be done in other areas for this to happen but those are out of the scope of the post.

Of course, there is no simple, one-size-fits-all solution, but I wanted to share our approach with everyone, and hopefully, if you encounter a similar situation this could be of use.

If you have any thoughts or questions, please share, and I will be happy to answer in the comments.
