+++
title = 'Delayed job, Heroku and sending an email with attachments'
date = 2014-04-21
tags = ["rails", "ruby", "heroku", "delayed_job"]
+++

Rails make it extremely easy to send emails. I'm not going to explain how to do it; there are pretty good tutorials around the internet,
this one is well explained: [Action Mailer Basics](http://edgeguides.rubyonrails.org/action_mailer_basics.html).


It is common to delay the task in our rails applications. That way, the app doesn't wait for emails to be sent to users. Many good gems help us,
I usually use [delayed_job](https://github.com/collectiveidea/delayed_job). This gems makes it easy, we have to prepend
`delay` in our process ` Mailer.delay.sendMail(args)`.


Some of the emails we send in my current project have attached files. Usually, for that type of task, it is common to create a temp file that stores the information,
and send it as arguments to mailer action.


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

That's one way of doing it, but [Heroku](https://www.heroku.com) do not allow us to write a file in the system due to the file system they have.
There are multiple solutions to this problem. We will explore using [S3](http://aws.amazon.com/) to store the file and reference in the email.

Once the file is uploaded to the S3, it can be downloaded and attached to the email.

So how we could do this:

The gem `aws-sdk` can help to interact with the S3 service.

Let's create the file and write whatever we want; I made some helper methods to generate and store a file in S3.

```ruby
# app/controller/helpers.rb
# We must write this to use the S3 store file
require 'aws-sdk'

def create_file
# This way the file will be created and close when the block is finish
  file = File.open("#{Rails.root}/tmp/filename.txt", "w+") do |f|
    f.write ("Hello")
    f.write ("\n")
    f.write ("World")
  end
  file
end

def store_S3(file)
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

We could call our Mailer with our file URL to read from and send the email inside our controller.

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

I hope this helps anyone; Heroku has problems if the file is too big. I'm working to solve it, and I hope to get an answer and share it with all of you.

Happy coding.







