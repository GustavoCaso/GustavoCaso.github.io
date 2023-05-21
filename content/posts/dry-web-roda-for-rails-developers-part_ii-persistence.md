+++
title = 'Dry-web-roda for Rails Developers Part II (Persistence)'
date = 2017-07-17
tags = ["ruby", "dry-rb", "ORM", "rom-rb"]
+++

Following my previous post, [Dry-web-roda part 1](/posts/dry-web-roda-for-rails-developers_part_i/), I have decided to create my small website to keep track of all the things I learn throughout the day.

Yes, I know, another `Today I Learned Website` 😓 - [til_web](https://github.com/GustavoCaso/til_web), but this time I started with the persistence layer, and I wanted to share my experience with you.

First, we need [dry-web-roda](https://github.com/dry-rb/dry-web-roda) to behave as our web application stack. When creating a new project with it, we have two options regarding our application's architecture point of view: `umbrella` or `flat`.

**Umbrella** means that our functionality will be divided into sub-apps - for example, the public and admin sites.

**Flat** is a simpler architecture with a single module for the entire app.

In my app, I decided to use the `flat` option for simplicity at the beginning, but in the future, I plan to move to sub-apps.

Let's start by creating our new application by running the new command:

```
dry-web-roda new til_web --arch=flat
```

Today, we will focus mainly on the persistence layer.

To start, we must create our new development database; note that the name will be extracted from the project's name (in this case, `til_web_development`). All the `ENV` information is in the `.env` file at the project's root.

Currently, we only support PostgreSQL (gem pg) as the database storage. If you don't have it installed, I find it really easy to use the [postgres app](https://postgresapp.com/) to reduce the burden.

To create the database we can use some of the commands that the Postgres app installed for us:

```
create_db -h localhost til_web_development`
```

Let's continue creating some migrations files to set up our database schema. `dry-web-roda` uses [rom-rb](http://rom-rb.org/) as its persistence toolkit. This allows us to have a clean separation of responsibilities and make an app that remains easy to change.

We can use the rake task provided to create a new migration file:

```
bundle exec rake db:create_migration[add_author]
```

The command will create a new file inside our `db/migrate` folder.

Rom uses [sequel](https://github.com/jeremyevans/sequel) for the database migration engine.

For any aspect related to migrations, please refer to the [Sequel documentation](http://sequel.jeremyevans.net/rdoc/files/doc/schema_modification_rdoc.html)

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

Running `bundle exec rake db:migrate` will create our new table.

Now that we have created a database table, we can start making some [Relations](http://rom-rb.org/learn/sql/relations/), [Commands](http://rom-rb.org/learn/sql/commands/) and [Repositories](http://rom-rb.org/learn/repositories/quick-start/) all of this concepts belongs to [rom-rb](http://rom-rb.org/).

Inside our `lib/persistence` folder we have `relations` and `commands` folders.

Relations are the interface to a particular collection in our data source, which is either a table or a view in SQL. We could think of them as our models.

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

We must define a [schema](http://rom-rb.org/learn/core/schemas/) and some composable, reusable query methods to return filtered results from our database table.

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

Commands are used to write to our database. By default ROM comes with `create`, `update` and `delete`, but you can create your custom ones by following the [commands guide](http://rom-rb.org/learn/advanced/commands/).

Finally, let's create our `Repository`. The repository works as the primary interface to interact with our database.

For my project, I created the folder `repositories` inside the `til_web`, which would use [auto_register](http://dry-rb.org/gems/dry-system/container/) from `dry-system` to register them in my container.

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

Our last step is to create some sample data to play with it in the console.

We open our `sample_data.rb` file and change it to:

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

We use `bundle exec rake db:sample_data` to populate the database.

Now accessing the console by typing `bin/console` allows us to check that everything has been stored in the database.

To access the repository, we type `TilWeb::Conatiner['repositories.tils']`, which will return an instance of `TilWeb::Repositories::Tils` with all the dependencies that it needs.

Lastly, we can check that there is data in the database by writing `TilWeb::Container['repositories.tils'][1]`

```
> TilWeb::Container['repositories.tils'][1]
=> #<ROM::Struct::Til id=1 title="Illo qui laborum dolores." text="Illum laboriosam adipisci incidunt. Ad aliquam ratione non adipisci quia velit. Veritatis eum minus ut quod mollitia sit. Ea tenetur aliquam fugit mollitia. Rerum ratione et dignissimos a et enim necessitatibus. Animi nesciunt qui rerum voluptatem ipsum atque ad.">
```

And that's it 🎉.

I know some concepts are different than what we are used to working with, and I could keep talking about them, but this post is getting quite long, so I will stop here for now.

If you have any thoughts or questions, please share, and I'll be happy to answer in the comments.

Also, here are some extra resources regarding `rom-rb` at it's benefits.

* [A conversational introduction to rom-rb](https://www.icelab.com.au/notes/a-conversational-introduction-to-rom-rb)

* [Conversational rom-rb, part 2: types, associations, and update commands](https://www.icelab.com.au/notes/conversational-rom-rb-part-2-types-associations-and-update-commands)

Thank you for reading this far!
