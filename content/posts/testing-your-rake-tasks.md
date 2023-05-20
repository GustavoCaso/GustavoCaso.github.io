+++
title = 'Testing rake tasks'
date = 2015-11-11
tags = ["rake", "ruby", "rspec"]
+++

This post continues our work in [Improving our rake tasks with OOP](/posts/improving-our-rake-tasks/).

In this one, we will discuss how to test our `rake` task; the example will be very straightforward.
We will invoke the `rake` task and expect that some classes `receive` the correct arguments.

I will use [Rspec](http://rspec.info/) as my test framework.

And I will continue with the same example from the last post.

```ruby
require 'rake'

describe 'wunderground_daily rake task' do
  before(:each) do
    load File.expand_path('tasks/wunderground_daily.rake', __FILE__)
    Rake::Task.define_task(:environment)
  end

  after(:each) do
    Rake::Task.clear
  end

  context 'wunderground_daily:check_flights' do
    it 'call WeatherInformation with correct class and method' do
      expect(WeatherInformation).to receive(:get_weather_information).with(Pathify::Flight, :airport_destination)
      Rake::Task['wunderground_daily:check_flights'].invoke
    end
  end

  context 'wunderground_daily:check_trains' do
    it 'call WeatherInformation with correct class and method' do
      expect(WeatherInformation).to receive(:get_weather_information).with(Pathify::TrainReservation, :station_destination)
      Rake::Task['wunderground_daily:check_trains'].invoke
    end
  end
end
```

So the first we do is load `rake`, and we create a `before` block, which will be executed before every test.
We load in memory the `file` where our tasks live, so the rake will know what task to `execute.` And we seat up the **environment**

The `after` block took some time to figure out why, but I was getting failing the tests because I was missing this one. Rake will store all the task that has been assigned, so the test where sharing the result, or so the expectation wasn't met.

There is this great documentation for `Rake`; I really encourage you to look it [Rake](http://ruby-doc.org/stdlib-1.9.3/libdoc/rake/rdoc/index.html)

Finally, in our test, we must `invoke our task` to execute it and see what results we get.

Also, because we previously refactored our `rake,` this test is easy to implement.
I will continue this series of posts with a more complete one, focusing on testing.

Thanks for reading; if you have any comments, please don't hesitate.


