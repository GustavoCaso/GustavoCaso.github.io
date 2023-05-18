+++
title = 'Rails Postgresql Error ORDER clause'
date = 2014-04-14
tags = ["rails ", " postgresql"]
+++

Postgresql ORDER clause or be used in a aggregate function

While working in a rails project recently, I step on an error that say: `PG::Error: ERROR:  column "number" must appear in the GROUP BY clause or be used in an aggregate function`,
I didn't have any idea how to solve it.



Reading for a while I think I have a small understand, why this error appear.

We have to specify which column should be use for the `GROUP BY` or for the `ORDER` inside our `SELECT` for example `Sale.all.select(:ordered_on, :number).group(:ordered_on, :number).order(:number)`.

With this query , we are able to retrieve all sales group by `ordered_on` and `number` and order by it as well.

This error jump while using Postgresql because it is more restrictive than MySQL. So remember be careful if you are deploying to Heroku because it's default database.

Hope this help you.

Here are some good references for the future. [Postgresql Tutorial](http://www.postgresqltutorial.com/)

