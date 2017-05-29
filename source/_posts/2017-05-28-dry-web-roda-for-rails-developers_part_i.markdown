---
layout: post
title: "Dry-web-roda for Rails Developers Part I"
date: 2017-05-28 20:14:45 +0200
comments: true
categories: [ruby, dry-rb]
---

Lately, I have been playing around and contributing to the great ecosystem of [dry-rb](http://dry-rb.org/),
first of all, I have to say the community is absolutely fantastic, super supportive and eager to welcome many new contributors.

First I will like to thank [Piotr Solnica](https://github.com/solnic), [Andy Holland](https://github.com/AMHOL), [Tim Riley](https://github.com/timriley) and [Nikita Shilnikov](https://github.com/flash-gordon) they have been really helpful and patience with my many questions.

Yesterday I decided to start playing around with a gem call [dry-web-roda](https://github.com/dry-rb/dry-web-roda) this small framework aim to provide an alternative to building web apps using ruby, with the use of small libraries such as [dry-view](https://github.com/dry-rb/dry-view), [dry-container](https://github.com/dry-rb/dry-container), [dry-transaction](https://github.com/dry-rb/dry-transaction), [roda](https://github.com/jeremyevans/roda), [rom](https://github.com/rom-rb/rom) and many more the help you build clearer, flexible and more maintainable code.

<!-- more -->

The post tries to create a bridge for [Rails](http://rubyonrails.org/) Developers and encourage them to try this new alternatives, that will bring joy and a fresh concepts for building web apps with ruby.

After installing the gem we can create a sample app by typing `dry-web-roda new github_stalker --arch=flat` this will create the file structure.

```
├── bin
├── config
├── db
├── lib
│   ├── github_stalker
│   │   └── views
│   └── persistence
│       ├── commands
│       └── relations
├── log
├── spec
│   └── support
│       └── db
├── system
│   ├── boot
│   └── github_stalker
├── transactions
└── web
    ├── routes
    └── templates
        └── layouts
```

At first the structure is quite different from what we are use to in a typical Rails app, but I will try my best to explain it.

First the system folder is where all the configuration lives, we can think of them as our initializers, this will be the entry point of our application. There are many small libraries involve for making everything work, that I can not go into detail in just one post, I will try cover all of them in a series of posts.
```
├── system
    ├── boot
    │   ├── monitor.rb
    │   ├── rom.rb
    │   └── view.rb
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
But to start with one of the files we can start with `application.rb` this file contains all the configuration regarding `routes`, `container` and plugins to be use with `roda`.

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

What is a container I'm going to bring the words from the official website `dry-container is a simple, thread-safe container, intended to be one half of the implementation of an IoC container` or how I understand it a container “gives you access to the objects that make up your application”.

Roda is the router is a Routing Tree Web Toolkit I will not go into much detail since I really new to it.

Following is the `container.rb` and `import.rb` which in my opinion is where all the magic happens, thanks to `dry-conatiner` and `dry-auto_inject`. This holds the configuration for loading the files for our application, more or less like `auto_load_path` of Rails but only using ruby methods and variables `require` and `$LOAD_PATH`.

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

At first site we see some configuration for name and namespace and some `auto_register`, this will register in your container all the files inside our `lib/github_stalker` folder following the convention from the file structure, so for example

```
├── lib
    ├── github_stalker
        ├── github
        │   ├── client.rb # will register inside our container under the name github.client
        │   ├── fetch_gists.rb # github.fetch_gists
        │   └── fetch_info.rb # github.fetch_info
        ├── users
        └── validate_input.rb # users.validate_input
```

And thanks to `dry-auto_inject` all this files will be accessible. But only if we need them, making really efficient by only including what we need.

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

Well I think this post is getting to long and I don't want to take more of your time. Thank you for reading it I will keep creating more posts explaining the rest of the structure and libraries involve in `dry-rb`, lastly I really encourage all ruby developers to try the libraries from `dry-rb` they are great and bring a fresh view in the ruby world.

All the example were taken from an example app I built using `dry-web-roda` if you want to check the code please follow [repo](https://github.com/GustavoCaso/dry-web-roda-example)

Please any doubt or thought please shared I'll be happy to answer in the comments.
