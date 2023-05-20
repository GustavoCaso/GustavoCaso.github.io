+++
title = 'Dry-web-roda for Rails Developers Part I'
date = 2017-05-28
tags = ["ruby", "dry-rb"]
+++

**Revised by Piotr Solnica and Andy Holland**

Lately, I have been playing around and contributing to the extraordinary ecosystem of [dry-rb](http://dry-rb.org/).
The community is absolutely fantastic, supportive, and eager to welcome many new contributors.

First, I will like to thank [Piotr Solnica](https://github.com/solnic), [Andy Holland](https://github.com/AMHOL), [Tim Riley](https://github.com/timriley) and [Nikita Shilnikov](https://github.com/flash-gordon) - they have been helpful and patient with my many questions.

Yesterday I started playing with a gem called [dry-web-roda](https://github.com/dry-rb/dry-web-roda). This small framework aims to provide an alternative to building web apps using ruby, using small libraries such as [dry-view](https://github.com/dry-rb/dry-view), [dry-container](https://github.com/dry-rb/dry-container), [dry-transaction](https://github.com/dry-rb/dry-transaction), [roda](https://github.com/jeremyevans/roda), [rom](https://github.com/rom-rb/rom) and many more; they help you build more transparent, flexible and more maintainable code.


The post tries to create a bridge for [Rails](http://rubyonrails.org/) developers and encourages them to try alternatives, bringing joy and fresh concepts for building web apps with Ruby.

After installing the gem, we can create a sample app by typing `dry-web-roda new github_stalker --arch=flat`- this will create the file structure.

```
├── bin
├── config
├── db
├── lib
│   ├── github_stalker
│   │   └── views
│   └── persistence
│       ├── commands
│       └── relations
├── log
├── spec
│   └── support
│       └── db
├── system
│   ├── boot
│   └── github_stalker
├── transactions
└── web
    ├── routes
    └── templates
        └── layouts
```

At first, the structure is quite different from what we are used to in a typical Rails app, but I will explain it.

First, the system folder is where all the configuration lives. We can think of them as our initializers. Many small libraries are involved in making everything work, and I can not go into detail in just one post. I will cover all of them in a series of posts.

```
├── system
    ├── boot
    │   ├── monitor.rb
    │   ├── rom.rb
    │   └── view.rb
    ├── boot.rb
    └── github_stalker
        ├── application.rb
        ├── container.rb
        ├── import.rb
        ├── repository.rb
        ├── settings.rb
        ├── transactions.rb
        ├── view_context.rb
        └── view_controller.rb
```

We will start with `application.rb`. This file contains all the configuration regarding `routes`, `container` and plugins to be used with `roda`.

```ruby
require "dry/web/roda/application"
require_relative "container"

module GithubStalker
  class Application < Dry::Web::Roda::Application
    configure do |config|
      config.container = Container
      config.routes = "web/routes".freeze
    end

    opts[:root] = Pathname(__FILE__).join("../..").realpath.dirname

    use Rack::Session::Cookie, key: "github_stalker.session", secret: GithubStalker::Container.settings.session_secret

    plugin :csrf, raise: true
    plugin :flash
    plugin :dry_view

    route do |r|
      r.multi_route

      r.root do
        r.view "welcome"
      end
    end

    load_routes!
  end
end
```

What is a container? I'm going to bring the words from the official website `dry-container is a simple, thread-safe container, intended to be one half of the implementation of an IoC container` or how I understand it, a container "gives you access to the objects that make up your application".

Roda (the router) is a Routing Tree Web Toolkit.

Following is the `container.rb` and `import.rb`, which in my opinion, is where all the magic happens, thanks to `dry-container` and `dry-auto_inject`. These files hold the configuration for loading the files for our application, more or less like `auto_load_path` of Rails.

```ruby
require "dry/web/umbrella"
require_relative "settings"

module GithubStalker
  class Container < Dry::Web::Umbrella
    configure do
      config.name = :github_stalker
      config.default_namespace = "github_stalker"
      config.settings_loader = GithubStalker::Settings
      config.listeners = true

      config.auto_register = %w[
        lib/github_stalker
      ]
    end

    load_paths! "lib", "system"

    def self.settings
      config.settings
    end
  end
end
```

At first sight, we see some configuration for name, namespace and some `auto_register`. This will register all the files inside our `lib/github_stalker` folder  in your container, following the convention from the file structure, so for example

```
├── lib
    ├── github_stalker
        ├── github
        │   ├── client.rb # will register inside our container under the name github.client
        │   ├── fetch_gists.rb # github.fetch_gists
        │   └── fetch_info.rb # github.fetch_info
        └── users
            └── validate_input.rb # users.validate_input
```

And thanks to `dry-auto_inject`, all these files will be lazily loaded as required, making it efficient.

```ruby
require "github_stalker/import"

module GithubStalker
  module Github
    class FetchGists
      include GithubStalker::Import['github.client'] # at the instance level we will have access to `client`

      def call(input)
        client.gists(input)
      end
    end
  end
end
```

This post is getting too long, and I don't want to take more of your time. Thank you for reading it. I will keep creating more posts explaining the rest of the structure and libraries involved in `dry-rb` - they are great and bring a new view in the ruby world.

All the above examples were taken from an example app I built using `dry-web-roda`; if you want to check the code, check this [repo](https://github.com/GustavoCaso/dry-web-roda-example)

If you have any thoughts or questions, please share, and I'll be happy to answer in the comments.
