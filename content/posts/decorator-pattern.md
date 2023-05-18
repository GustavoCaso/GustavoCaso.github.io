+++
title = 'Decorator Pattern'
date = 2015-02-28
tags = ["ruby", " rails"]
+++

Last night I was reading some blogs about ruby patterns, for me is sometimes difficult to understand the use of this kind of patterns, mostly because in my
work we just throw thousand of lines of code, and deal with a lot of legacy code from other person who contribute to our projects. I currently work for comparason website in Spain [Kelisto](https://www.kelisto.es)
Lately there is one projects that is architecture is strongly inheritance, so there are many classes that inherited from a parent class.



There are sometimes were there is no difference between then except for class name.

Reading about this [Decorator](http://en.wikipedia.org/wiki/Decorator_pattern) where you are able to dynamically add more funcionality to your objects, keep me thinking is there any way I can create the same structure but instead with thousands of classes with Decorators.


So imagine a simple class Card with all it's methods, validations etc...

```ruby
  class Card
    def description
      "#{name}:#{self.class}"
    end

    def total_tax
      (year_fess + extra_charges / 5) *100
    end
  end
```

Imagine a bunch of methods more.

One day the stakeholder comes and says to you, we are going to have a  new type of card. Quickly you think easy I just create a new type of card that inherited, and easy as pie we have our new card type.
So basically have the same methods, or eventually we will add some methods that are the same for both, or just uniq depending of the specifications.

So I thought maybe I can use this pattern to create the same behavior, but without that many classes or models if you prefer. Using the Decorator pattern I can create some modules with
custom behavior for my card class and then added to them. This way I dont create a new class just super charge my class with this methods.

```ruby
  module CreditCardModule
    def total_tax
      super + credit_card_fees
    end

    def description
      "#{name}: CreditCard"
    end
  end
```

This way I won't have a lot of class/models but one card class that I will dynamically add more functionality

```ruby
class Card
  include CreditCardModule
  .
  .
  .
end
```ruby

##### Decorators not always the best solution
This pattern is not always the best solution of course, what if the credit card have some fields that are only for this type of card, in that case inheritance is a good option to have in mind.
With that in mind there are other solutions that could come more efficient. I will continue this line of thought in future publications.









