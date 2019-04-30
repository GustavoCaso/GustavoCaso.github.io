---
layout: post
title: Migrating millions of keys without downtime
date: 2019-04-30 01:43 UTC
tags: [ruby, redis]
published: false
---

Last year in September I joined the Job Patterns team at Shopify. The mission of the team is to provide a stable platform so developers can write their background jobs that power one of the biggest e-commerce platforms in the world.

Since I joined, I have been gathering context around the various components that create a unique Shopify ecosystem.  This post is going to focus on how we managed to migrate millions of keys from one of our Redis instances to another without downtime or any incidents.

To provide some context, Shopify is a massive monolith Ruby on Rails application and the background job architecture consist of ActiveJob, Resque, and Redis. Besides the functionality that those libraries provide by default, we have created many additional modules that allow developers to define custom behaviour for their jobs: Locking, LockQueue, Concurrency, Retry, Status and many more.

The module we'll be discussing here is the Locking module.  A module where the logic is quite simple: Before enqueuing the job, it checks for the existence of the lock key if the lock key does not exist we acquire it until the job is done executing and finally we release the lock. This functionality allows for only one job with the same arguments to be executed at the same time.

At the current growth rate of Shopify, we are looking into multiple ways to improve the background jobs infrastructure to scale with the high demand of the platform. That is why we are going to allow developers to enqueue and pop jobs from multiple Redis instances so we can distribute the load. Of course, these changes need to be transparent for the application developers and merchants.

Ok, time to explain why we have to migrate so many keys. At Shopify, we have multiple Redis instances: resque, disposable, carts, locks, etc... When we introduced the Locking functionality we decided that lock keys for the jobs belong to resque responsibility, so it made sense to store those keys in the resque Redis instance, that solution worked for us until we decided to distribute the load of enqueue and dequeue of jobs. However, if we are going to enqueue jobs in a round robin fashion into multiple Redis, we need to know exactly where the lock keys are stored to ensure acquiring and releasing works.

Our best solution is to move the keys from the resque instance to the lock instance. How did we manage to do it without any downtime and any service disruption on the platform?

On a system which doesn't process X jobs a minute, we could stop the application and move the keys from one Redis to the other and deploy the code changes, so everything works as expected unfortunately or fortunately ( 😄 ) for us that is not the case.

We came with a 3 step plan that would allow us to do it. All steps required code changes on the application, so the full migration took roughly 2 weeks.

First lets introduce our Locking module, this is going to be a simplify version of the one currently maintain at Shopify:

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


**First Step:**

We modify the `locked?` function of the locking module to first check on the locks Redis and then on the resque Redis. With this change, the functionality stays the same, but we introduce the locks redis as a new dependency.

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

We are going to start `acquiring` on the locks redis.
The `release` function will check first on the Resque Redis and if it wasn't successful it will try releasing on the Locks redis.
The `locked?` function stays the same as in the first step.

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
  [redis, lock_redis]
end
```

You could notice above we switch the order of the `redises` function, this is to avoid having to many redis requests, since old the lock keys are stored in the resque Redis only after deploying this change the new locks will be acquire in the locks redis.

__Note:__ Ideally we will like to stay in this step of the migration as short as possible, that way we reduce the numbers of Redis requests.

**Last Step:**

We change all the code to make sure that the only redis instance involves with the Locking module is the locks redis. All acquiring, releasing and checking actions of the keys have migrated over.


```ruby
private

def redis
  Lock.redis
end
```


With these steps, we were able to migrate the lock keys successfully without affecting our SLO or any SLO of the other teams 🎉 🎉

Of course, the changes weren't as straightforward as described above, there were other components involved, many tests to modify and some infrastructure changes to be done in other for this to happen but those are out of the scope of the post.

After the migration we encountered new challenges: Would be the locks redis be able to handle the load? Is the locks redis a single point of failure?

Of course, there is no simple solution or a solution that fits all, but I wanted to share with everyone the process, and hopefully, if you encounter a similar situation this could come of use.
