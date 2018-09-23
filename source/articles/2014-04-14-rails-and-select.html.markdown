---
layout: post
title: "Rails and select"
date: 2014-04-14 12:10:45 +0200
tags: [ruby, rails]
---

In a rails project I'm working on I was trying to select from the database some sales with some conditions.

To start the `Sale` model has multiples associations .

```ruby
class Sale < ActiveRecord::Base
  belongs_to :voucher
  belongs_to :client
  has_many :line_items, dependent: :destroy
  has_many :bills
end
```

So the goal of this select was to obtain all the `Sales` where the `LineItems` has express_checkout set it to true.

READMORE

I thought it was easy, in the `Sales controller` I will get all the sales `Sales.all` do a select `Sales.all.select` that's the easy part, inside the block is were
I got lost, because I have access to sale not the line_items associated to it, I started trying to fetch the line_items and perform another
select inside.

```ruby
Sales.all.select do |sale|
  sale.line_items.select do |line_item|
    line_item.express_checkout == true
  end
end
```
So I see this code and thought that must be right, but I keep getting all the `Sales`.

A friend of mine told to extract a method for this kinf of task in the `Sales`model, so I gave it a try, and created a new method call express_checkout?.
This method did basically the same but instead I store the result of the select in a variable and then check if there where any object inside that variable.
That approach worked.

So my thoughts were inside the select always will return an array with the elemnts that passed from the condition, but I didn't know we have to store them in a variable and then check it.

The final code:

```ruby
def express_production?
  l = line_items.select{|line_item| line_item.express_checkout == true}
  l.any?
end
```

