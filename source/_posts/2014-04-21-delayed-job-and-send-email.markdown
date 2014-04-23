---
layout: post
title: "Delayed job and Send email with Attachements"
date: 2014-04-21 20:21:46 +0200
comments: true
categories: [rails, ruby]
---

##Sending emails with rails

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

Thats one way of doing it, the problem with delay job is that the temp file will be erase before the the email process is finish,
after some reading and understanding this problem I decided to change my approach and create a file inside our `temp` folder inside our `Rails.root`
so when sending it there is a file to read from.


```ruby
File.open("#{Rails.root}/tmp/new_file.csv", "w*") do |f|
  f.write ("Hello")
  f.write ("\n")
  f.write ("World")
end


Mailer.delay.sendMail


```

Inside our Mailer the new code is:
```ruby
class Mailer < ActionMailer::Base

  def send_mail
    attachments['filename.csv'] = File.read("#{Rails.root}/tmp/new_file.csv")
    mail(to: foo@bar.com, subject: 'Welcome to My Awesome Site')
    File.delete("#{Rails.root}/tmp/new_file.csv")
  end

end
```

I know deleting the file inside the send_mail action is not the best way of doing it, I'm trying to work with hooks provided by `delayed_job` gem, but right I haven't figured out.
When I learn how to do it I will update the post.

Happy coding.







