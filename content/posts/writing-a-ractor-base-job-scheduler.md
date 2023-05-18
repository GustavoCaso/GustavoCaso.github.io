+++
title = 'Writing a Ractor Base Job Scheduler'
date = 2020-09-19
tags = ["ruby", " concurrency"]
+++

I was reading a fantastic [article](https://kirshatrov.com/2020/09/08/ruby-ractor-web-server/) from my colleague Kir, and I could help myself to feel nerd snipped by:

> For those curious to try Ractor, I'd suggest to try implementing other things that benefit from parallel execution, for instance a background job processor.

### What is a background job processor?

A background job processor allows you to offload heavy computational tasks from the main process to other processes.

Imagine a typical request/response inside a Rails app, you would like to provide your users with the fastest response, but some requests have operations associated with it that could derail the response. A good example could be sending emails.

Is very common in the Rails community to offload those task into a background job processor.

The most common libraries used for that are: [sidekiq](https://github.com/mperham/sidekiq), [resque](https://github.com/resque/resque), and [sucker_punch](https://github.com/brandonhilkert/sucker_punch)

### Let's build a background job processor using Ractor

Before continuing reading, I recommend having a look at [Ractor](https://github.com/ko1/ruby/blob/dc7f421bbb129a7288fade62afe581279f4d06cd/doc/ractor.md#shareable-objects) documentation.

Here is a simple diagram of our background job processor.

```
                                   +                            +----+
                                   |       Separate Process     |    |
                                   |                            | W  |
+----------------+                 |     +--------------------+ +----+
|                |                 |     |                    | +----+
|   Main Process |                 |     |                    | |    |
|   (Rails app)  |                 |     |   Job Processor    | | W  |
|                |                 |     |                    | +----+
+-------+--------+                 |     |   Takes work from  | +----+
        |                          |     |   the queue and    | |    |
        |              +---------+ |     |   send it to the   | | W  |
        |  Push        |         | |     |   workers          | +----+
        +------------> |  Queue  +------>+                    | +----+
                       +---------+ |     |                    | |    |
                                   |     |                    | | W  |
                                   |     |                    | +----+
                                   |     |                    | +----+
                                   |     +--------------------+ |    |
                                   |                            | W  |
                                   |       W = Worker           +----+
                                   |
                                   |
                                   +

```


For the `Queue` part, we are going to use [redis](https://redis.io/).

Our main process:

```ruby
QUEUE_NAME = "jobs:queue:low"
redis = Redis.new

redis.rpush(QUEUE_NAME, new_job)
```

The main process will write work into the queue, and our background job processor will dequeue that work and execute it.

Let's start building our job processor.

```ruby
redis = Redis.new
loop do
  puts "Waiting on work"
  queue, job = redis.brpop(QUEUE_NAME)
  puts "Pushing work from #{queue} to available workers"
  pipe.send(job, move: true)
end
```

You can see here that we have an endless loop.
The idea of the job processor is to check for work in the queue (`QUEUE_NAME`)

The command [BRPOP](https://redis.io/commands/brpop) blocks until there is data in the queue. Once it finds something in the queue, it passes to the `pipe`.

#### What is a pipe?


```ruby
pipe = Ractor.new do
  loop do
    Ractor.yield(Ractor.recv, move: true)
  end
end
```

I like to think of it as [IPC](https://en.wikipedia.org/wiki/Inter-process_communication) (Inter-process communication) but with Ractors.

One Ractor push something, and another Ractor takes it.

```
+----------------+                                 +-----------------+
|                |        +--------------+         |                 |
|                |  push  |              |  take   |                 |
|  Ractor 1      +-------->+     Pipe     +------->+   Ractor 2      |
|                |        |              |         |                 |
|                |        +--------------+         |                 |
+----------------+                                 +-----------------+
```

The `pipe` is going to allows us to communicate between the job processor and the workers.

Lastly, we have our set of workers that are going to be waiting for the job processor to signal when them when there is work to do.

```ruby
NUMBER_OF_WORKERS = 10

workers = NUMBER_OF_WORKERS.times.map do |index|
  worker = Worker.new(index)
  Ractor.new(pipe, worker) do |pipe, worker|
    loop do
      # this is a blocking operation
      # This Ractor will wait until there is a something to take from the pipe
      job = pipe.take
      puts "taken job from pipe by #{Ractor.current} and work it by worker #{worker.id}"

      worker.work(job)
    end
  end
end
```

I omitted the code for the `Worker` class but will include everything at the end of the article.

Here is the complete job scheduler code:

```ruby
pipe = Ractor.new do
  loop do
    Ractor.yield(Ractor.recv, move: true)
  end
end

workers = 4.times.map do |index|
  worker = Worker.new(index)
  Ractor.new(pipe, worker) do |pipe, worker|
    loop do
      job = pipe.take
      puts "taken job from pipe by #{Ractor.current} and work it by worker #{worker.id}"

      worker.work(job)
    end
  end
end

redis = Redis.new
loop do
  puts "Waiting on work"
  queue, job = redis.brpop(QUEUE_NAME)
  puts "Pushing work from #{queue} to available workers"
  pipe.send(job, move: true)
end
```

It couldn't get more straightforward than that.

### Limitations with Ractor:

I tried to encapsulate the main loop inside a Ractor, but due to the sharing limitations of Ractor, I wasn't able.

```ruby
job_server = Ractor.new(pipe, QUEUE_NAME) do |pipe, queue_name|
  redis = Redis.new

  loop do
    puts "Waiting on work"
    queue, job = redis.brpop(queue_name)
    puts "Pushing work from #{queue} to available workers"
    pipe.send(job, move: true)
  end
end

loop do
  Ractor.listen(job_server, *workers)
end
```

I got the error:

```ruby
thrown by remote Ractor. (Ractor::RemoteError)
  from /Users/gustavocaso/Desktop/ractor.rb:100:in `block in <main>'
  from /Users/gustavocaso/Desktop/ractor.rb:99:in `loop'
  from /Users/gustavocaso/Desktop/ractor.rb:99:in `<main>'
/usr/local/lib/ruby/gems/3.0.0/gems/redis-4.2.2/lib/redis/client.rb:406:in `_parse_options': can not access non-sharable objects in constant Redis::Client::DEFAULTS by non-main ractor. (NameError)
```

The [Redis::Client::DEFAULTS](https://github.com/redis/redis-rb/blob/master/lib/redis/client.rb#L9-L27)  constant is not [sharable](https://github.com/ko1/ruby/blob/dc7f421bbb129a7288fade62afe581279f4d06cd/doc/ractor.md#shareable-objects) inside Ractors code

I had a similar issue when trying to use the ruby JSON library. I had to use [oj](https://github.com/ohler55/oj) to parse the job after dequeuing from Redis.

### Conclusions

Even with the sharing limitations of Ractor, I'm very excited about the new possibilities that it will enable to ruby developers.

As Kir's said in his article, I would encourage anyone to experiment with it.

Here is the complete [gist](https://gist.github.com/GustavoCaso/f6b14360ec1e4031f438c51045ee2d64) of the code so you can experiment with it.

