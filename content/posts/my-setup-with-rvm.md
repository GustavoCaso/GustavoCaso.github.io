+++
title = 'My Setup with rvm, gemsets and bundler'
date = 2016-10-17
tags = ["ruby", "rvm"]
+++

It took me some time to figure out how to correctly set up every project I worked on, not that I work on tons of projects, but now that I have a better understanding of how it makes more sense.

## RVM

So [RVM](https://rvm.io/) or Ruby Version Manager allows you to install different Ruby version on the same machine without them collision it.


When RVM installs a new version of ruby, it will create a new folder inside the `~/.rvm/gems` folder with the name of the ruby version. That way, when installing different ruby versions have their folder containing::

```bash
drwxr-xr-x  11 ………  staff  374 Sep 13 23:10 .
drwxr-xr-x  13 ………  staff  442 Oct 17 15:37.
drwxr-xr-x   5 ………  staff  170 Sep 13 23:10 bin
drwxr-xr-x   2 ………  staff   68 Sep 13 23:10 build_info
drwxr-xr-x   4 ………  staff  136 Sep 13 23:10 cache
drwxr-xr-x   2 ………  staff   68 Sep 13 23:10 doc
-rw-r--r--   1 ………  staff  468 Jul 30 17:45 environment
drwxr-xr-x   2 ………  staff   68 Sep 13 23:10 extensions
drwxr-xr-x   4 ………  staff  136 Sep 13 23:10 gems
drwxr-xr-x   4 ………  staff  136 Sep 13 23:10 specifications
drwxr-xr-x  13 ………  staff  442 Sep 13 23:10 wrappers
```

When using that version of ruby, all the gems installed via `gem install ...` will be installed in the `gems` folder; that way, we can have different versions of the same gem in the different ruby versions on our computer.

RVM allows us to specify which ruby version we use when `cd` inside a project, with a special file `.ruby-version`. This file will have the ruby version; you can create it manually or tell RVM to do it with this command
`rvm --create --`ruby-version ruby-2.3.3`.

### Gemsets

Having the same ruby version for multiple projects could create conflicts because we have a different version of `nokogiri` in both projects. To avoid conflict errors, we have to use `bundle exec ...` to use the version specified by the `Gemfile.`

We can avoid this tedious extra typing by creating a different gemset for each project, and within each gemset set, all the gems the project needs. So every time we create a new `gemset`, a new folder is added to `~/.rvm/gems/#{ruby-version}@#{gemset}`, so with this, we altogether avoid conflicts.

Another good thing is when finishing working on the project. We can delete all the gems from that project just by deleting the gemset by typing `rvm gemset delete #{gemset}.`

As well as the ruby version, RVM can automatically use a gemset by specifying with the file `.ruby-gemset`

### Workflow

When I start on a new project, I create a ruby version and a gemset file and install all the gems I need for that project.

There is an excellent command that will create both files for us, let's imagine the ruby version is 2.3.3, and the project is called test_rvm, so I want to create a gemset for this project.

Running  `rvm use --create ruby-2.3.3@test_rvm`

Another cool thing is that for every ruby version, RVM automatically creates a global gemset, which all gem sets for that ruby version will share, so before creating a new gemset for that ruby version, I make sure that I have installed `bundler.`

```
rvm use global
gem install bundler
```

This way, I don't have to install it in every gemset.

Do you have any idea on how to improve my workflow? Please let me know.
