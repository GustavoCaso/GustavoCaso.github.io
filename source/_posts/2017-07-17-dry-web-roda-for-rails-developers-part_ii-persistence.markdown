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

Let's start by creating our new application.

Running the new command `$ dry-web-roda new til_web --arch=flat` - it will generate the file structure.

Today will focus mostly in the persistence part.

To start we have to create our new development database; note that the name will be extracted from the name of the project (in this case `til_web_development`). All the information regarding the `ENV` is located in the `.env` in the root of the project.

At the moment we only support postgres (gem pg) as database storage. If you don't have it installed I find super easy the [postgres app](https://postgresapp.com/) to lift up the burden of doing it.

To create the database we can use some of the commands that postgres app install for us, `$ create_db -h localhost til_web_development`.

Let's continue by creating some migrations files; `dry-web-roda` use [rom-rb](http://rom-rb.org/) as his ORM that allow us to have a clean separation of responsibilities and make an app that remains easy to change.

We can use the rake task provided to create a new migration file -
`$ bundle exec rake db:create_migration[add_author]`, will create a new file inside our `db/migrate` folder.

Rom use [sequel](https://github.com/jeremyevans/sequel) as the database migration engine.

For any aspect related to migrations, please refer to [Sequel documentation](http://sequel.jeremyevans.net/rdoc/files/doc/schema_modification_rdoc.html)

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

Running `$ bundle exec rake db:migrate` will create our new table.

Now we are going we can start creating some [Relations](http://rom-rb.org/learn/sql/relations/), [Commands](http://rom-rb.org/learn/sql/commands/) and [Repositories](http://rom-rb.org/learn/repositories/quick-start/) all of this concepts belongs to [rom-rb](http://rom-rb.org/)

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

We need to define a [schema](http://rom-rb.org/learn/core/schemas/) and some composable, reusable query methods to return filtered results from our database table. For example `by_id(id)`.

Let's continue by creating a new command.

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

Commands are use to write to our database, by default ROM comes with `create`, `update` and `delete`, you can create your custom ones [commands guide](http://rom-rb.org/learn/advanced/commands/).

And finally lets create our `Repository`. Repository works as the main interface to interact with our Database.

For my project I created the folder `repositories` inside the `til_web`, that would use [auto_register](http://dry-rb.org/gems/dry-system/container/) from `dry-system` to register them in my container.



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
# need the application to be booted in order to access the container.
require_relative '../system/boot'
require 'faker'


def create_til(attrs)
  # this line is use to access the rom container, that is created at the booting process
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

To populate the database we use `$ bundle exec rake db:sample_data`.

Now accessing the `console`, by typing `$ bin/console`. Allow us to check that everything has been storage in the database.

To access the repository we type `TilWeb::Conatiner['repositories.tils']` that will return an instance of `TilWeb::Repositories::Tils` with all the dependencies that it needs.

And lastly we can check that there are data in the database by writing `TilWeb::Container['repositories.tils'][1]`
```
> TilWeb::Container['repositories.tils'][1]
=> #<ROM::Struct::Til id=1 title="Illo qui laborum dolores." text="Illum laboriosam adipisci incidunt. Ad aliquam ratione non adipisci quia velit. Veritatis eum minus ut quod mollitia sit. Ea tenetur aliquam fugit mollitia. Rerum ratione et dignissimos a et enim necessitatibus. Animi nesciunt qui rerum voluptatem ipsum atque ad.">
```

At thats it 🎉.

I know some concepts are quite different than what we are use to work with, and I could keep talking about them, but the post is getting quite long. Thanks so much for reading.

If you have any thoughts or questions, please share and I’ll be happy to answer in the comments.

Also here are some extra resources, regarding rom-rb at it's benefits.

* [A conversational introduction to rom-rb](https://www.icelab.com.au/notes/a-conversational-introduction-to-rom-rb)

* [Conversational rom-rb, part 2: types, associations, and update commands](https://www.icelab.com.au/notes/conversational-rom-rb-part-2-types-associations-and-update-commands)

Once again Thank you for reading.
