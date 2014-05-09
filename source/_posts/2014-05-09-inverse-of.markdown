---
layout: post
title: "Inverse_of ActiveRecord::Base"
date: 2014-05-09 17:15:21 +0200
comments: true
categories: [Active Record, rails]
---

While reading the documentation of [ActiveRecord::Associations](http://api.rubyonrails.org/classes/ActiveRecord/Associations/ClassMethods.html), there are many useful methods, that I didn´t know about it.
But one came to me as quite useful, `inverse_of`. This method allow us to work with associations that haven`t been save yet, so it will work with the memory.
This will optimise our object loading.

I will so you with some example.
```ruby
Class Post < ActiveRecord::Base
  has_many :comments, inverse_of :post
end

Class Comment < ActiveRecord::Base
  belongs_to :post, inverse_of :comments
end
```
<!-- more -->

Setting like this our models will allow to do some nice thing in our console.

```ruby
p = Post.create
=> #<Post id: nil, name: nil >
c = p.comments.create
=> #<Comment id: nil, body: nil >
c.post
=> #<Post id: nil, name: nil >
```
Without the inverse the last line will return `nil` because is trying to hit the database but there isn't any record save in it
