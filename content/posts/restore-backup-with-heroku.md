+++
title = 'Restore PostgreSQL backup within Heroku'
date = 2014-04-23
tags = ["heroku", "postgresql"]
+++

Recently I was playing around with my local database in my Rails project deployed using [Heroku](https://www.heroku.com). Eventually, I screwed up and had to delete all the data.
I use `rake db: reset and `rake db:create`, but how would I populate the data again?

With three simple lines of code in your terminal, you can download your production database and load it in your local environment.

In your terminal and your project folder, type:

```
heroku pgbackups:capture
```

```
curl -o latest.dump `heroku pgbackups:url`
```

These two lines create a backup and save it to the file called `latest.dump`

```
pg_restore --verbose --clean --no-acl --no-owner -h localhost -U my user -d mydb latest.dump
```
With this last line, we are restoring the database from the backup file; remember to change `myuser` for your current user and `mydb` for your database name.

I hope this helps.

