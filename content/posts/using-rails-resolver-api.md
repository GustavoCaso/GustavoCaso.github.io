+++
title = 'Using Rails Resolver Api'
date = 2015-05-04
tags = ["rails"]
+++

Yesterday at work, we decided to improve our user experience in web mobile.

We decided it was time to create a different view for each action. In [Rails 4](http://guides.rubyonrails.org/) there is this really cool feature called [Variants](http://guides.rubyonrails.org/4_1_release_notes.html#action-pack-variants)

Depending on the variant you set `phone || tablet` usually from a `before_action` inside your `ApplicationController`, rails will be smart enough to render the proper layout and the right view,
only with the condition that you add the variants to the file name `show.html+mobile.erb`.

Our problem is that we have a complex Rails 3.2 application with many dependencies, which is quite difficult to upgrade to Rails 4.1.


So I implemented a simple but effective version of the variant feature.

At first, I was lost on implementing this in the best way possible. My first approach was to create a simple method, `render_template`, that was added to every controller action. The method will decide which template to use based on the `request.user_agent`. Of course, there were better ideas than that one. I had to write a lot of `respond_to` blocks all over the application.

One friend of mine pointed me in the right direction. Rails Resolvers, also to this great book [Crafting Rails Appliactions](https://pragprog.com/book/jvrails2/crafting-rails-4-applications).

The book has many tutorials for building plugins to extend rails functionality.

Ok, so armed with my book and enthusiasm, I decided to try it again.

I had to create a simple class called 'MobileViewResolver' with just two methods.
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

The initialize method accepts two parameters: the path to look for the templates and a pattern. In this case, I use the one for rails 4.1, including the variants.

The `find_templates` method receives the template name, a prefix (i.e. the controller path), a boolean marking if the template is a partial or not and a hash with details.

We modify the details hash to include the mobile variant, and Rails will look for that pattern inside the app/views folder. If no file is found, it will fall back to the default ones.

We only have to include the new resolver in our controller.

```
prepend_view_paths MobileViewResolver.new
```

And that is all.

We manage to add this new feature with just a straightforward class.

Rails has some awesome APIs waiting to be used.




