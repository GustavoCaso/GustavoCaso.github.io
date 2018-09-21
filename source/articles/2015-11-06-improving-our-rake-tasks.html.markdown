---
layout: post
title: "Improving our rake tasks"
date: 2015-11-06 14:25:42 +0100
comments: true
tags: [ruby, rake]
---

Lately I have been writting some `rake` tasks, for downloading backups, for accesing API's, or for automating teadious and repetitive work. Rake task are great, but dangerous at the same time.

We tend to add so much code to our rake task, that they become a source of errors.
Following the principles of OOP we can clean our `rake` tasks, improving our code and making them much easier to test.

READMORE

Let's look up an example from work:


```ruby

namespace :wunderground_daily do
  desc 'Get all flights on 10 days before depart and get weather for airports'
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
              airport_code.create_weather_observation(forecastday, fl)
            end
          end
        end
      end
    end
  end
end
```

As you can see the rake task is pretty big and involve multiple things.

* Connecting to an external Service [Wunderground](http://www.wunderground.com/)
* Fetching valid records
* Updating record with the data from the response

This type of code is difficult to read and difficult to test, so extracting behaviour to different classes could improve the readability and scalability for the future.

Usually in my process of refactoring code involve multiple steps, extracting logic to method, moving that method to a class and test. In a perfect world the test are written first but the world is not as perfect as wanted.

So following the steps I extract the logic into a method.

```ruby
namespace :wunderground_daily do
  desc 'Get all flights on 10 days before depart and get weather for airports'
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
              airport_code.create_weather_observation(forecastday, fl)
            end
          end
        end
    end
  end
end
```

So far so good, the rake task has reduce the size to just one line and all the logic is inside the new created method `get_weather_information`. But this is difficult to test and still this new method is doing to many things.

So we continue our path, and create a new class that will help with our rake task.
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
        destination.create_weather_observation(forecastday, fl)
      end
    end
  end
end
```

This new class `WeatherInformation` will improve our work by allowing us to test the different methods and we are following the Single Responsability Principle.

So we are able to get the `segments` the current `weather_information` of our segments and `update` the information if needed.

As you can see each action has his own method, so is easier the change.

Right now I'm pretty happy with the result, there is always room for improvment, we could move the call to the external api to a `background job`, but that is out of the scope of this read.

Extracting this behaviour to a new class will help us in the future, if the Product Manager decided that we need `weather information` for our `Train` model.

```ruby
namespace :wunderground_daily do
  desc 'get weather information for our Models'
  task get_info: :environment do
    WeatherInformation.new(Flight, :airport_destination).get_weather_information
    WeatherInformation.new(Train, :airport_destination).get_weather_information
  end
ends
```

This post is getting a little long, I have the intention the talk about testing the rake task, but I will leave that one for the second part.

Thanks for the read. Please if you have any idea for improve just comment and I will answer as soon as possible.
