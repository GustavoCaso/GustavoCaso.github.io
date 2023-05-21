+++
title = 'Lazy Enumerator'
date = 2014-07-03
tags = ["ruby"]
+++

Recently reading some posts about ruby, I found myself with this exciting method called `lazy`.
This method will help us when looping over some objects: Array, Hashes, and Files.
If you want to loop through a huge array, and we only want to select the first 20 elements that evaluate our criteria, this is the kind of job for our `lazy` method.


# Without the use of lazy

``` ruby
arr =  (1..1000).to_a
selected = []
arr.select do |x|
  selected << x if x.odd?
  break if selected.size > 20
end
p selected
```

# With the use of lazy method
```ruby
arr =  (1..1000).to_a
selected = arr.lazy.select do |x|
  x if x.odd?
end.take(20).force
p selected
```

The syntax is more straightforward and is better in performance.
