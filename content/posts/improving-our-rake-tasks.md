+++
title = 'Improving our rake tasks with OOP'
date = 2015-11-06
tags = ["ruby", "rake"]
+++

I have been writing some `rake` tasks for downloading backups, accessing APIs, or automating tedious and repetitive work. Rake tasks are great, but dangerous at the same time.

We add so much code to our rake tasks that they become a source of errors.
Following the principles of OOP, we can clean our `rake` tasks, improving our code and making them much easier to test.

Let's look up an example from work:

```ruby

namespace :wunderground_daily do
  desc 'Get all flights on 10 days before depart and get weather for airports.'
  task check_flights: :environment do
    w_api = Wunderground.new(secret) #o_O

    flights = Flight.future.lt(utc_arrival_date: (Time.now.utc + 10.days))
    flights.each do |fl|
      if airport_code = fl.airport_destination
        # If there are any flight without weather or has flight in 2 days, update info
        wo = airport_code.weather_observations.where(date: fl.utc_arrival_date.beginning_of_day)
        if wo.empty? || fl.utc_arrival_date.beginning_of_day == (Date.today + 2.days) && !wo.last.updated_at.today?

          puts fl.id
          response = w_api.forecast10day_for("#{airport_code.latitude},#{airport_code.longitude}")
          calls_count += 1

          puts "#{airport_code.latitude},#{airport_code.longitude}"
          if response["forecast"]
            response["forecast"]["simpleforecast"]["forecastday"].each do |forecastday|
              airport_code.create_weather_observation(forecast day, fl)
            end
          end
        end
      end
    end
  end
end
```

As you can see, the rake task is big and involves multiple things.

* Connecting to an external Service [Wunderground](http://www.wunderground.com/)
* Fetching valid records
* Updating record with the data from the response

This type of code is difficult to read and test, so extracting behaviour to different classes could improve the readability and scalability for the future.

Usually, my process of refactoring code involves the following:
Multiple steps.
Extracting logic to a method.
Moving that method to a class and testing.

So following the steps, I extract the logic into a method.

```ruby
namespace :wunderground_daily do
  desc 'Get all flights on 10 days before depart and get the weather for airports.'
  task check_flights: :environment do
    get_weather_information(Flight, :airport_destination)
  end

  def get_weather_information(model, target)
    w_api = Wunderground.new(secret) #o_O
    segments = model.future.lt(utc_arrival_date: (Time.now.utc + 10.days))
    segments.each_with_index do |sg, i|
      if destination = sg.send(target)
        # If there are any train without weather or has train in 2 days, update info
        wo = destination.weather_observations.where(date: sg.utc_arrival_date.beginning_of_day)
        if wo.empty? || sg.utc_arrival_date.beginning_of_day == (Date.today + 3.days) && !wo.last.updated_at.today?
          response = w_api.forecast10day_for("#{airport_code.latitude},#{airport_code.longitude}")
          if response["forecast"]
            response["forecast"]["simpleforecast"]["forecastday"].each do |forecastday|
              airport_code.create_weather_observation(forecast day, fl)
            end
          end
        end
    end
  end
end
```

So far, so good; the rake task has reduced the size to just one line, and all the logic is inside the newly created method `get_weather_information`. But this isn't easy to test, and this new method is still doing too many things.

So we continue our path and create a new class to help with our rake task.

```ruby
class WeatherInformation
  attr_reader :model, :target

  def initialize(model, target)
    @model = model
    @target = target
  end

  def get_segments
    model.future.lt(utc_arrival_date: (Time.now.utc + 10.days))
  end

  def get_weather_observations(destination)
    destination.weather_observations.where(date: sg.utc_arrival_date.beginning_of_day)
  end

  def get_information(model, target)
    segments = get_segments

    segments.each_with_index do |sg, i|
      if destination = sg.send(target)
        # If there are any train without weather or has train in 2 days, update info
        wo = get_weather_observations(destination)
        if wo.empty? || sg.utc_arrival_date.beginning_of_day == (Date.today + 3.days) && !wo.last.updated_at.today?
          update_weather_destination(destination)
        end
      end
    end
  end

  def update_weather_destination(destination)
    w_api = Wunderground.new(secret) #o_O
    response = w_api.forecast10day_for("#{destination.latitude},#{destination.longitude}")
    if response["forecast"]
      response["forecast"]["simpleforecast"]["forecastday"].each do |forecastday|
        destination.create_weather_observation(forecast day, fl)
      end
    end
  end
end
```

This new class, `WeatherInformation,` will improve our work by allowing us to test the different methods, and we are following the Single Responsibility Principle.

So we can get the `segments` and the current `weather_information` of our segments and `update` the information if needed.

As you can see, each action has its method, so the change is easier.

Right now, I'm pretty happy with the result; there is always room for improvement; we could move the call to the external API to a `background job`, but that is out of the scope of this read.

Extracting this behaviour to a new class will help if the Product Manager decides we need `weather information` for our `Train` model.

```ruby
namespace :wunderground_daily do
  desc 'get weather information for our Models'
  task get_info: :environment do
    WeatherInformation.new(Flight, :airport_destination).get_weather_information
    WeatherInformation.new(Train, :airport_destination).get_weather_information
  end
ends
```

This post is getting a little long, I intend to talk about testing the rake task, but I will leave that one for the second part.

Thanks for the read. If you have any ideas for improvement, comment, and I will answer as soon as possible.

