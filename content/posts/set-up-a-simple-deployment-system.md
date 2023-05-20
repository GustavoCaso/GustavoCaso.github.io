Before deploying to production is always recommended to test everything first; you probably have a Test suit to test your application, but you never know what will happen when you deploy all changes to production.

People usually create a staging environment that copies the production environment.

You deploy first to staging and test everything, so after checking everything, you may deploy to production with the confidence that everything works perfectly.

Heroku allows you to create multiple environments; Heroku has excellent documentation and community posts.
This post shows us how to create different environments. [Managing Multiple Environments for an App](https://devcenter.heroku.com/articles/multiple-environments)

If you have an existing App, how to add a new environment needs to be clarified, but I will help you.

From our terminal, we must have [Heroku Toolbelt](https://toolbelt.heroku.com/) installed to allow us to use the command line.
Heroku has a method called [fork](https://devcenter.heroku.com/articles/fork-app) that will clone an existing app into another; you don't need to create an app first; Heroku will create the app and clone everything into this new app.

```
heroku fork -a your-app-name your-app-name-staging
heroku fork -a your-app-name your-app-name-staging --region eu # there the possibility to pass a region option
```

Replace `your-app-name` with the actual name of your app and `your-app-name-staging` with the name of the staging environment you want.

This process might take some time.

After that, we must add a new remote to git to push to it.

```
git add remote staging git@heroku.com:your-app-name-staging.git
```
Now we can push to staging `git push staging master`.

After that, you may go further. You can install a gem [Paratrooper](https://github.com/mattpolito/paratrooper) that helps us with the deployment task.

We have to create a rake task inside our `lib/task` folder, call it `deploy.rake` and place this code.

```ruby
require 'paratrooper.'

namespace :deploy do
  desc "deploy to staging"
  task :staging do
    deployment = Paratropper::Deploy.new('your-app-name-staging,' tag: 'staging')

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

With this file in our app, we can call the rake tasks `rake deploy:staging` or `rake deploy:production`.

Inside the production task, we see:

```
deploy.match_tag = 'staging'
```

The line above would only allow us to push to production changes that have been pushed to staging before; this way, we must complete the staging deployment first.
