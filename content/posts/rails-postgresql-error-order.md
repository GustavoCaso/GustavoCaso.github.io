+++
title = 'Rails Postgresql Error ORDER clause'
date = 2014-04-14
tags = ["rails ", "postgresql"]
+++

While working on a rails project recently, I stepped on an error that said: `PG::Error`: ERROR:  column "number" must appear in the GROUP BY clause or be used in an aggregate function`,

I didn't have any idea how to solve it.

After reading for a while, I understand why this error appears.

We must specify which column should be used for the `GROUP `BY` or the `ORDER`` inside our `SELECT, for example, `Sale`.all.select(:ordered_on, :number).group(:ordered_on, :number).order(:number)`.

With this query, we can retrieve all sales groups by `ordered_on`` and `number` and order by it.

The error was raised while using Postgresql because it is more restrictive than MySQL. So remember to be careful if you are deploying to Heroku because it's the default database.

I hope this helps you.

Here are some excellent references for the future. [Postgresql Tutorial](http://www.postgresqltutorial.com/)

