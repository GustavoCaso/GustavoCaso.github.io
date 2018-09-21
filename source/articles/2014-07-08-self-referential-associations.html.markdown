---
layout: post
title: "Self-referential Associations"
date: 2014-07-08 17:12:53 +0200
comments: true
tags: rails, ruby, associations
---

When I first heard about this concept I was a little bit confuse. I was reading [Learn rails by Example](http://www.railstutorial.org/) by Michael Hart

READMORE

By that time the concept was to much for me, so I did some research.
I found Railscast episode [Self-referential Associations](http://railscasts.com/episodes/163-self-referential-association), it threw some
light in the concept, but I still confuse.

Right now I think I understand the concept a little bit better but there always room for improvement.
I'm going to show you the code for a simple self-referential association for friends and followers.


```ruby
class User < ActiveRecord::Base
  has_many :reviews
  has_many :queue_items, -> {order(:position)}
  has_many :friendships, foreign_key: :user_id
  has_many :followers, class_name: "Friendship", foreign_key: :friend_id

  has_secure_password validations: false
  validates_presence_of :full_name, :email

  def included_in_queue?(video)
    queue_items.include?(video)
  end
end
```

I have to say that Ryan Bates solution probably is more efficient than this one but for me this one looks more clear.

Ok, so we have a `User` model and a `Friendship` model that has `user_id` and `friend_id`.
Now is time to link this two models. The line `has_many :friendships, foreign_key: :user_id` is telling `ActiveRecord::Base` to get all
row from the `friendships table` where `user_id` and the `current_user.id` match.

This is the query:
```
SELECT "friendships".* FROM "friendships"  WHERE "friendships"."user_id" = ?  [["user_id", 1]]
```

Thanks to the method `has_many` and some of the options we are able to get what we want, the option `foreign_key` let us specify the foreign key used for the association.

The last line `has_many :followers, class_name: "Friendship", foreign_key: :friend_id` let us get the people that are following us.

Because there is no followers table we have to specify the class_name in this case Friendships and the foreign_key option will
help us specify the association.

This is the query:
```
SELECT "friendships".* FROM "friendships"  WHERE "friendships"."friend_id" = ?  [["friend_id", 1]]
```

I'm sure there are probably more better ways to do it but this one help me understand it better.
