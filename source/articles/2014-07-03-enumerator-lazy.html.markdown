---
layout: post
title: "Enumerator::Lazy"
date: 2014-07-03 18:47:22 +0200
comments: true
tags: ruby
---

Recently reading the some posts about ruby I found myself with this interesting method call `lazy`.
This method will help us when looping over some objects: Array, Hashes, Files ....... If you want to loop though an array that is huge, an we only want to select the first 20 element that evaluate our criteria, this is the kind of job for our `lazy` method.

READMORE

# Without Lazy

``` ruby
arr =  (1..1000).to_a
selected = []
arr.select do |x|
  selected << x if x.odd?
  break if selected.size > 20
end
p selected
```

This looks familiar but we may use the lazy method.

# With Lazy method
```ruby
arr =  (1..1000).to_a
selected = arr.lazy.select do |x|
  x if x.odd?
end.take(20).force
p selected
```

This way the syntax is more clear and is better in performance.
