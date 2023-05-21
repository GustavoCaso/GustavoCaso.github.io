+++
title = 'Self-referential Associations in Rails'
date = 2014-07-08
tags = ["rails", "ruby", "associations"]
+++

When I first heard about this concept, I needed clarification. I was reading [Learn Rails by Example](http://www.railstutorial.org/) by Michael Hart

The concept was too complex to understand by then, so I researched.
I found the Railscast episode [Self-referential Associations](http://railscasts.com/episodes/163-self-referential-association), which shed some light on the concept.

I will show you the code for a simple self-referential association for friends and followers.


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

Ryan Bates's solution is probably more efficient than this one, but this one looks clearer to me.
Ok, we have a `User` model and a `Friendship` model with `user_id` and `friend_id` columns.

Now is the time to link these two models. The line:

```
has_many :friendships, foreign_key: :user_id
```

Tells `ActiveRecord` to get all
rows from the `friendships` table where `friendships.user_id` and the `user.id` match.

Here is the raw SQL query
```
SELECT "friendships".* FROM "friendships"  WHERE "friendships"."user_id" = ? [["user_id", 1]]
```

Thanks to the method `has_many` and some options, we can get what we want.
The option `foreign_key` let us specify the foreign key used for the association.

The last line:
```
has_many :followers, class_name: "Friendship", foreign_key: :friend_id
```

Allos us to get the people following us.

Because there are no followers table, we have to specify the class_name in this case, Friendships, and the foreign_key option will
help us determine the association.

Here is the raw SQL query
```
SELECT "friendships".* FROM "friendships"  WHERE "friendships"."friend_id" = ? [["friend_id", 1]]
```

There are probably better ways to do it, but this one helps me understand it better.
