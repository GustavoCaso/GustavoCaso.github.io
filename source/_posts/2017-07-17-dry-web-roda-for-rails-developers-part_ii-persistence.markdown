---
layout: post
title: "Dry-web-roda for Rails Developers Part II (Persistence)"
date: 2017-07-17 07:38:28 +0200
comments: true
categories: [ruby, dry-rb, ORM, rom-rb]
---

Following with my previous post [Dry-web-roda part 1](http://gustavocaso.github.io/2017/05/dry-web-roda-for-rails-developers_part_i/) I decided to create my own small website for keeping track of all the things I learn through the day.

Yes I know another Today I Learned Website 😓 - [til_web](https://github.com/GustavoCaso/til_web).

This time I started by the persistence layer and I wanted to share with you my experience.

First of all we will need [dry-web-roda](https://github.com/dry-rb/dry-web-roda) to work as our web application stack,
when creating a new project with it, we have two options regarding the architecture point of view of our application: `umbrella` or `flat`.

Umbrella means that our functionality will be divided into sub-apps for example the public site and the admin site.
Flat is a simpler architecture, with a single module for the entire app.

In my app I decided to use the `flat` option for simplicity at the beginning, but in the future I plan to move to sub-apps.

<!-- more -->

Let's start by creating our new application `$ dry-web-roda new til_web --arch=flat` it will generate the file structure.

I will focus mostly in the persistence part.

To start we have to create our new development database, the name is extracted from the name of the project in this case `til_web_development` all the information regarding the `ENV` is located in the `.env` in the root of the project. All of the environment variables are loaded by `Dry::Web::Settings`

At the moment we only support `pg` postgres as database storage. I find super easy the [postgres app](https://postgresapp.com/) for installing and everything.

To create the database we can use some of the commands that postgres app install for us, `$ create_db -h localhost til_web_development` that simple.

Let's continue with our migrations files `dry-web-roda` use [rom-rb](http://rom-rb.org/) as his ORM that allow us to have a clean separation of responsibilities and make an app that remains easy to change.

For creating the migrations file we delegate it to rom-rb - `$ bundle exec rake db:create_migration[add_author]`, will create a new file inside our `db/migrate` folder.

Once we open the file:

```ruby
ROM::SQL.migration do
  change do
  end
end
```

 Rom use [sequel](https://github.com/jeremyevans/sequel) as the database migration engine. There is this great documentation about migrations [Sequel documentation](http://sequel.jeremyevans.net/rdoc/files/doc/schema_modification_rdoc.html)

 We continue by creating a table and adding some fields to it.

```ruby
 ROM::SQL.migration do
  change do
    create_table :tils do
      primary_key :id

      column :title, String, null: false
      column :text, 'text', null: false
      column :created_at, DateTime,  null: false, default: Sequel::CURRENT_TIMESTAMP
      column :updated_at, DateTime, null: false, default: Sequel::CURRENT_TIMESTAMP
    end
  end
end
```

 Once happy with our migration we can run `$ bundle exec rake db:migrate`, it will create our new table.

 Now we are reading to start looking at the file structure of our project and create some [Relations](http://rom-rb.org/learn/sql/relations/), [Commands](http://rom-rb.org/learn/sql/commands/) and [Repositories](http://rom-rb.org/learn/repositories/quick-start/)

 Inside our `lib/persistence` folder we have `relations` and `commands` folders.

 Relations are the interface to a particular collection in our data source, which in SQL terms is either a table or a view. We could think them as our models.

```ruby
 module Persistence
  module Relations
    class Tils < ROM::Relation[:sql]
      schema(:tils) do
        attribute :id, Types::Serial
        attribute :title, Types::Strict::String
        attribute :text, Types::Strict::String
      end

      def by_id(id)
        where(id: id)
      end
    end
  end
end
```

 We need to define a [schema](http://rom-rb.org/learn/core/schemas/) and some composable, reusable query methods to return filtered results from our database table `by_id` this methods will be use by our repositories.

 Them we create a new [Command](http://rom-rb.org/learn/advanced/commands/), inside our `commands` folder, they are use to interact with the database `Create`, `Update` and `Delete`.

```ruby
 module Persistence
  module Commands
    class CreateTil < ROM::Commands::Create[:sql]
      relation :tils
      register_as :create
      result :one
    end
  end
end
```

And the last step is to create our `Repository` it works as the main interface to interact with our Database, having them been a separate object, they ensure the rest of the app doesn't have any coupling to the implementation details of our data source.

For my project I created the folder `repositories` inside the `til_web`, that would be [auto_register](http://dry-rb.org/gems/dry-system/container/) thanks to `dry-system`. We can see that in our [container.rb](https://github.com/GustavoCaso/til_web/blob/master/system/til_web/container.rb#L9)


```ruby
require 'til_web/repository'

module TilWeb
  module Repositories
    class Tils < TilWeb::Repository[:tils]
      def [](id)
        tils.by_id(id).one!
      end
    end
  end
end
```

Our last step is to create some sample data so we can play with it in the `console`.

We open our `sample_data.rb` file.

```ruby
# need the application to be booted in order to access their container.
require_relative '../system/boot'
require 'faker'


def create_til(attrs)
  # this line is use to access the rom container, that is created at the botting process
  # of the application.
  # inside the `system/boot/rom.rb`
  TilWeb::Container['persistence.rom'].commands[:tils][:create].call(attrs)
end

20.times do
  create_til(
    title: Faker::Lorem.sentence(4),
    text: Faker::Lorem.paragraph(5)
  )
end
```

After running `$ bundle exec rake db:sample_data` we can check that everything has been created by stepping in the `console`, by typing `$ bin/console`

To access the repository we type `TilWeb::Conatiner['repositories.tils']` that will return an instance of `TilWeb::Repositories::Tils` with all its dependencies.

And lastly we can check that there are data in the database by writing `TilWeb::Container['repositories.tils'][1]` this is the method that we have defined above in our repository class.

```
> TilWeb::Container['repositories.tils'][1]
=> #<ROM::Struct::Til id=1 title="Illo qui laborum dolores." text="Illum laboriosam adipisci incidunt. Ad aliquam ratione non adipisci quia velit. Veritatis eum minus ut quod mollitia sit. Ea tenetur aliquam fugit mollitia. Rerum ratione et dignissimos a et enim necessitatibus. Animi nesciunt qui rerum voluptatem ipsum atque ad.">
```

At thats it 🎉.

I know some concepts are quite different than what we are use to work with, and I could keep talking about them, but the post is getting quite long. Thanks so much for reading.

If you have any thoughts or questions, please share and I’ll be happy to answer in the comments.

Also here are some extra resources.

* [A conversational introduction to rom-rb](https://www.icelab.com.au/notes/a-conversational-introduction-to-rom-rb)

* [Conversational rom-rb, part 2: types, associations, and update commands](https://www.icelab.com.au/notes/conversational-rom-rb-part-2-types-associations-and-update-commands)

Once again Thank you for reading.
