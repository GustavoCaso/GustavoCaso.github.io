+++
title = 'Using Rails Resolver Api'
date = 2015-05-04
tags = ["rails"]
+++

#####Implement Rails Variants in Rails 3.1

Yesterday at work we decided that we need to improve our user experience in web mobile.
So we decided that it was time to create a different view for each action. Lately in [Rails 4](http://guides.rubyonrails.org/) there is this really cool feature called [Variants](http://guides.rubyonrails.org/4_1_release_notes.html#action-pack-variants)
really neat one if you ask me.



Basically depending on the variant you set `phone || tablet` usually from an `before_action` inside your `ApplicationController`, rails will be smart enough to render the right layout and the right view,
only with te condition that you add the variants to the file name `show.html+mobile.erb`.

Ok so easy as pie. Our problem we have a complex rails 3.2 application with a lot of dependencies, that is quite difficult to upgrade to rails 4.1.


So I decided to implement a simple but effective variation of the variants `#to manny variants around :)`

At first I was quite lost how to implement this the best way possible, well my first approach was to create a simple method `render_template` that added to every controller action will decided with template to use base on the `request.user_agent`. Of course that wasn't the best idea but I have to write a lot of respond_to blocks all over te application.

One friend of mine point to right direction Rails Resolvers, also to this great book [Crafting Rails Appliactions](https://pragprog.com/book/jvrails2/crafting-rails-4-applications).

So basically there some awesome tutorials for building this plugins to extend rails functionality, and a lot of information about the new Api they have.

It's true that is not easy finding documentation for the internal classes around Rails like `ActionView::Resolver`.

Ok so armed with my book and my enthusiasm I decided to give it another try.

Basically want I had to do was create a simple class called ´MobileViewResolver´ with just two methods.
```ruby
class MobileViewresolver < ::ActionView::FileSystemResolver

    def initialize
      super('app/views', ":prefix/:action{.:locale,}{.:formats,}{+:variants,}{.:handlers,}")
    end

    def find_templates(name, prefix, partial, details)
      details['variants'] = ['mobile']
      super(name, prefix, partial, details)
    end

end
```

The initialize method accept two parameters the path where to look for the templates and a pattern, in this case I use the one for rails 4.1 including the variants.
The `find_templates` method receives the template name, a prefix (i.e. the controller path), a boolean marking if the template is a partial or not and a hash of details.
We modify the details hash to include in our case our mobile variant and rails will look for that pattern inside the app/views folder. If no file is found it will fallback to the default ones.
We only have to include in our controller  this new resolver `prepend_view_paths MobileViewResolver.new`.
And that is all.

We manage to add this new feature with just one simple class.

Rails has some awesome API waiting to be use.




