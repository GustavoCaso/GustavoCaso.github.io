---
layout: post
title: "Delayed job and Send email with Attachements"
date: 2014-04-21 20:21:46 +0200
comments: true
categories: [rails, ruby, heroku, delayed_job]
---

##Sending emails with rails and Heroku (Updated 2014-04-24)

Rails makes extremly easy to send emails, I'm not going to explain how to do it, there are pretty good tutorials around the internet,
this one is really well explain: [Action Mailer Basics](http://edgeguides.rubyonrails.org/action_mailer_basics.html).

It is common to delay the task in our rails applications, that way the app doesn't stop. There are many good gems that help us,
I usually use is [delayed_job](https://github.com/collectiveidea/delayed_job). This gems makes it really easy, we just have to prepend
`delay` in our process `Mailer.delay.sendMail(args)`.


In my actual project some of the mails we send, has attachments files, usually for that type of task it is common to create a temp file that store the information,
and send it as arguments to mailer action.

<!-- more -->

Inside our controller.
```ruby
temp_file = Tempfile.new("new_file.csv")

temp_file.write ("Hello")
temp_file.write ("\n")
temp_file.write ("World")

temp_file.rewind
temp_file.close

Mailer.sendMail(temp_file).deliver

temp_file.unlink
```

Inside our Mailer class code could be like this:
```ruby
class Mailer < ActionMailer::Base

  def send_mail(file)
    attachments['filename.csv'] = File.read(file.path)
    mail(to: foo@bar.com, subject: 'Welcome to My Awesome Site')
  end

end
```

Thats one way of doing it, but [Heroku](https://www.heroku.com) do not allow us to write file in the system, do to the file system they have.
There multiple solutions to this problem, basically we can store the file in one of the store service like [S3](http://aws.amazon.com/).
To use the S3 service we must add a gem to our Gemfile `gem 'aws-sdk'`

Once the file is uploaded to the S3 it will be posible to download it and attach it to the email.

So how we could do this:

Lets create the file and write what ever we want, in my case I created some helper method to create the file and store it in [S3](http://aws.amazon.com/).
Inside `app/controller/helpers`.

```ruby
# We must write this to use the S3 store file
require 'aws-sdk'

class create_file
# This way the file will be created and close when the block is finish
  file = File.open("#{Rails.root}/tmp/filename.txt", "w+") do |f|
          f.write ("Hello")
          f.write ("\n")
          f.write ("World")
        end
  file
end

class store_S3(file)
# We create a connection with amazon S3
  AWS.config(
      :access_key_id => ENV['AWS_ACCESS_KEY_ID'],
      :secret_access_key => ENV['AWS_SECRET_ACCESS_KEY']
    )
    s3 = AWS::S3.new
    bucket = s3.buckets[ENV['S3_BUCKET_NAME']]
    object = bucket.objects[File.basename(file)]
# the file is not the content of the file is the route
    object.write(:file => file)
# save the file and return an url to download it
    object.url_for(:read)
end
```

With this two method we are able to create and store the file in AWS, no is the easy part send the email.

Inside our controller we could call our mailer with our file url, to read from and send the email.

```ruby
class SaleController < ApplicationController::Base
  def send_email
    file = create_file
    url = store_S3(file)
    Mailer.delay.send_mail(url)
  end
end
```

And for the last touch, inside our Mailer

```ruby
class Mailer < ActionMailer::Base

  def send_mail(url)
    attachments['filename.csv'] = open(url).read
    mail(to: foo@bar.com, subject: 'Welcome to My Awesome Site')
  end

end
```

I hope this could help anyone, there are some problems with heroku if the file is to big there might be some problems, I'm working to solve it, hopefuly I could get and answer and share with all of you.


Happy coding.







