---
layout: post
title: "Exporting and Import Heroku DB"
date: 2014-04-23 12:21:39 +0200
comments: true
categories: [heroku, postgresql]
---

##Heroku is AWESOME!

Recently I was playing around with my local database in my rails project, eventually I screw up, I had to delete all the data.
I use `rake db:reset`and `rake db:create` but how I was going to get all the data again, manually ? No way.

[Heroku](https://www.heroku.com) is a great tool for deploying.

With three simple lines of code in your terminal you are able to download your production database and load it in your local envairoement.

In your terminal and in your project folder type:

```
$ heroku pgbackups:capture
$ curl -o latest.dump `heroku pgbackups:url`
```

This two line create a backup and read it to file call latest.dump.

```
$ pg_restore --verbose --clean --no-acl --no-owner -h localhost -U myuser -d mydb latest.dump
```
With this last line we are restoring the database from the backup file, remember change `myuser` for your actual user and `mydb` for your database name.

Hope this helps.

