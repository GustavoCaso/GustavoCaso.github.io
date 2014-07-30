---
layout: post
title: "Set up a simple deployment system"
date: 2014-07-30 11:04:20 +0200
comments: true
categories: [heroku, rails]
---

##Understand heroku environments

Before deploy in production is always recommended to test everything first, probably you have a Test suit for test your application, but you never know want is going to happen when you deploy all changes to production.
What people usually do is create a staging environment, that copies the production environment.

Basically you deploy first to staging and then test evrything, so after checking everything you may deploy to production with the confidence that everything works perfectly.
Heroku as I said before is amazing it allow's us to create multiple environments, Heroku provides us with many documentation to create our environments.
[Managing Multiple Environments for an App](https://devcenter.heroku.com/articles/multiple-environments) This post show us how to create different environments.
The problem is if you have an existing App is not very clear how to do it, but I shall help you do it.

<!-- more -->

From our terminal, we must have [Heroku Toolbelt](https://toolbelt.heroku.com/) installed, to allow us use the command line.
Heroku has method call [fork](https://devcenter.heroku.com/articles/fork-app) that will clone an existing app into another, you don't need to create app the first, heroku will create the app and clone everything into this new app.
```
$ heroku fork -a your-app-name your-app-name-staging
$ heroku fork -a your-app-name your-app-name-staging --region eu # there the posibility to pass a region option
```
Replace `your-app-name` with the actual name of your app and `your-app-name-staging` with the name for the staging environment you want.
This process might take some time.

After that we must add a new remote to git so we can push to it.
```
$ git add remote staging git@heroku.com:your-app-name-staging.git
```
Now we can push to staging `git push staging master`

After that you may go a little further you can install a gem [Paratrooper](https://github.com/mattpolito/paratrooper) that help us with the deployment task.

The basic use of this gem is pretty simple we have to create a rake task inside our `lib/task` folder call it `deploy.rake` and place this code.
```ruby
require 'paratrooper'

namespace :deploy do
  desc "deploy to staging"
  task :staging do
    deployment = Paratropper::Deploy.new('your-app-name-staging', tag: 'staging')

    deployment.deploy
  end

  desc  "deploy to production"
  task :production do
    deployment = Paratrooper::Deploy.new('your-app-name') do |deploy|
      deploy.tag = 'production'
      deploy.match_tag = 'staging'
    end

    deployment.deploy
  end
end
```

With this file in our app we can call the different rake tasks `rake deploy:staging` or `rake deploy:production`.

Inside the production task is `deploy.match_tag = 'staging'`. This signifies that we only will push to production, what has been push to staging, this way we can not skip the staging part.


