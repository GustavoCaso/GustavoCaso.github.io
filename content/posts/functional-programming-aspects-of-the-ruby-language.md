+++
title = 'Functional programming aspects of the Ruby language'
date = 2017-11-03
tags = ["ruby", "functional programming"]
+++

**Revised by Andy Holland**

Lately, there has been a significant change in the industry towards functional programming; new languages have appeared: Elixir, Scala, and Elm.

With all this new hype, I decided to try some of them, and they are cool, but whether I like it at work, I mostly use ruby.

After playing with new functional programming languages, I noticed that Ruby has some similarities with functional programming.

I'm not going to say that ruby is an FP language nor that Matz, the creator is incorrect.

To quote Matz's words from an [interview](http://www.linuxdevcenter.com/pub/a/linux/2001/11/29/ruby.html) with O'Reilly.
> I wanted a scripting language that was more powerful than Perl and more object-oriented than Python.


So Ruby is an object-oriented language - that's for sure - but I wanted to share my idea that Ruby is a multipurpose language.

According to [Wikipedia](https://en.wikipedia.org/wiki/Functional_programming), for a programming language to be considered Functional, it needs to have the following:
- First-class and higher-order functions
- Pure functions
- Recursion
- Strict versus non-strict evaluation
- Type systems
- Currying
- Lazy Evaluation

We are going to explore some of the concepts with Ruby :gem:

### Higher-order functions
- takes one or more functions as arguments
- returns a function as its result

In ruby there are examples everywhere showing that ruby supports Higher-order functions.

Let's look at a well-known ruby method, `each.`

```ruby
[1,2,3,4].each { |x|  puts x*x } #=> [1,4,9,16]
```

We can consider the block as passing a function as an argument.

To clarify the case, you can write the same example using lambdas.

```ruby
square = -> { |x| puts x*x }
[1,2,3,4].each(&square) #=> [1,4,9,16]
```

Many ruby-core methods take a block or function as an argument `each,` `map,` `select.`

As for functions that return other parts, as not typical in Ruby, we can still do them.

```ruby
def adder(a, b)
  lambda { a + b }
end

adder_fn = adder(1, 2)
adder_fn.call # => 3
```
### First Class Functions and Support for Lambdas
Ruby has support for multiple types of functions, lambdas, and procs; there are shuttle differences between them, and we will need a whole post about it, but instead, there are many significant resources out there explaining the differences [articles](http://awaxman11.github.io/blog/2013/08/05/what-is-the-difference-between-a-block/).
How I like to think about them is as functions that can you stored in variables:

```ruby
add = -> (x,y) { x + y }
multiply = -> (x,y) { x * y }
```

One interesting pattern is the policy pattern that allows you to have a function execute the same tasks. Still, depending on the business logic, the callbacks for success or failure can be configured:

```ruby
def create_record(attributes, success_policy, failure_policy)
  if Record.create(attributes)
    success_policy.call
  else
    failure_policy.call
  end
end

success_policy = -> () { send_email }
failure_policy = -> () { puts 'something went wrong' }
create_record({name: 'John'}, succes_policy, failure_policy)
```
One thing that is not that common in ruby is storing method inside variables; using the method `method`, we can store them and pass it to functions as if they were a proc or a lambda.

An excellent video from [RubyTapas](https://www.rubytapas.com/2012/10/17/episode-011-method-and-message/) covers this topic very well.

For now, I will give you a small example:

```ruby
def hello
  yield
end

def hi
  "hi there."
end

hello &method(:hi)
```

### Currying
Currying means partially applying a function; here are a couple of definitions:

- Partial function application is calling a function with some number of arguments to get a function back that will take that many fewer arguments.
- Currying is taking a function that takes n arguments and splitting it into n functions that take one argument.

In Ruby, it is rare to use this technique, but this allows us to have small and reusable functions that can be easy to use and understand while you read your code.

Since Ruby 1.9, the Proc class has the method: `#curry`, which allows us to implement both options.

Let's see some examples.

```ruby
db_operation = lambda do |db_adapter, operation, record|
  db_adapter.send(operation, record)
end

create_db_operation = db_operation.curry(DB, :create)
delete_db_operation = db_operation.curry(DB, :delete)

create_db_operation.(maria_record)
delete_db_operation.(maria_record)
```

A nice plus is that working with these small functions, your code becomes much easier to test.

### Lazy evaluation
First of all, I encourage you to check this [blog post](http://patshaughnessy.net/2013/4/3/ruby-2-0-works-hard-so-you-can-be-lazy)  from Pat Shaughnessy

Lazy evaluation allows us to consume data on demand instead of evaluating everything up front.

Let's look at some examples:

```ruby
range = 1..Float::INFINITY
range.each { |x| x*x }.take(10) #=> endless loop!

# Using lazy

range = 1..Float::INFINITY
range.lazy.each { |x| x*x }.take(10) #=> [1,4,9,16,25,336,49,64,81,100]
```

Without the `lazy`, it tries to consume all the elements and transform them, ending in an endless loop, but the second example uses the lazy evaluation allowing it to collect only the data that it need.

This technique is advantageous when dealing with an enormous amount of data, and we do not want to transform all of them at once but only on demand.

### Type system

We all know that ruby is a dynamic language with great introspection capabilities and doesn't have a type system.

One of the reasons everyone loves ruby is due to these specifications, where we are allowed to do want we want. But having that freedom of choice, we tend to end with profoundly entangled and error-prone applications.

I'm not saying not to use ruby, but a type of system would allow us to have warranties of want sort of data we are working with would be fantastic.

[Dry-types](http://dry-rb.org/gems/dry-types/) to the rescue \o/.

Talking about `dry-types` would be out of scope for this post and probably requires one itself :wink: but just to let you know: if you are looking for a valuable option for your application, there is one.

### Pure functions

A pure function is a function which:

- Given the same input, it will always return the same output.
- Produces no side effects.

Having pure functions will give us a lot of benefits; the first one that comes to my mind is parallel processing.

Pure functions are also easy to understand, refactor and move around, making our application less error-prone.

And going to show a straightforward example.

```ruby
def double(number)
  number * 2
end
```

No matter what, if we introduce the same input, it will return the same output.

Ruby allow us to have mutable state, so it is up to us to change how we write our function in a pure way.

If you have any thoughts or questions, please share, and I'll be happy to answer in the comments.
