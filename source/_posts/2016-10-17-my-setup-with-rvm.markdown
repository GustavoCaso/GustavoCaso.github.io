---
layout: post
title: "My Setup with rvm, gemsets and bundler"
date: 2016-10-17 20:01:03 +0200
comments: true
categories: [ruby, workflow, rvm]
---

## My setup for every project
I took me some time, to figure out how to correctly setup every project I worked on, not that I work in tons of projects, but now that I have a better understanding of how to setup, it makes more sense to me, at least.

I'm sure I did not understand it early, because of my lack, in reading the documentation, I know is a horrible habit, I'm working on it.

### RVM

So [RVM](https://rvm.io/) or Ruby Version Manager, allow to install different ruby version on the same machine without them collision it.

<!-- more -->

When RVM installs a new version of ruby it will create a new folder inside the `~/.rvm/gems` folder with the name of the ruby version, that way when installing different ruby version each of them will have their own folder, each folder containing the next structure
```bash
drwxr-xr-x  11 ………  staff  374 Sep 13 23:10 .
drwxr-xr-x  13 ………  staff  442 Oct 17 15:37 ..
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
When using that version of ruby, all the gems installed via `gem install ...` will be installed in the `gems` folder, that way we can have different version of the same gem in the different ruby version, in our computer.

RVM allow to specify which ruby version are we using when `cd` inside a project, with a special file `.ruby-version`. This file will have the ruby version, you can create it manually or tell rvm to do it for you with this command
`rvm --create --ruby-version ruby-2.3.3` this will create the `.ruby-version` file.

#### Gemsets

When having the same ruby version for multiple projects, could create some conflicts, because we have a different version of `nokogiri` in both projects, to avoid conflict error, we have to use `bundle exec ...` to use the version specify by the `Gemfile`.

We can avoid this tedious extra typing by creating a different gemset for each project, and within each gemset all the gems that the project need. So every time we create a new `gemset` a new folder is added to `~/.rvm/gems/#{ruby-version}@#{gemset}` so with this we completely avoid conflicts.

Another good thing is when finishing working on the project, we can delete all the gems from that project just by deleting the gemset completely by typing `rvm gemset delete #{gemset}`

As well as the ruby version, RVM can automatically use a gemset by specifying with the file `.ruby-gemset`

## Workflow

When I start on a new project, I create a ruby-version and a ruby-gemset file, and install all the gems I need for that project.

There is a great command that will create both files for us, let's imagine the ruby version is 2.3.3 and the project is called test_rvm, so I would like to create a gemset for this project, just by typing `rvm use --create ruby-2.3.3@test_rvm`

Another cool thing is that for every ruby version RVM automatically created a global gemset, which all gemsets for that ruby version will share, so before creating a new gemset for that ruby version I make sure that I have installed `bundler`.

```
rvm use global
gem install bundler
```

This way I don't have to install in every gemset.

So this is my normal workflow, any idea of how to improve it, or if I'm doing something wrong. Please just let me know.
