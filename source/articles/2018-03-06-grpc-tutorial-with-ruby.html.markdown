---
layout: post
title: "Grpc Tutorial with ruby"
date: 2018-03-06 11:10:20 +0100
comments: true
tags: [ruby, grpc]
---

So the other day I found an exciting project [Anycable](http://anycable.io/) that allow using custom WebSocket server within your ruby application. I immediately got hooked up, and I started reading about it, and the first thing that I never heard of it was [Grpc](https://grpc.io).

READMORE

Grpc is an Open Source RPC framework developed by Google which uses [protocol buffers](https://developers.google.com/protocol-buffers/docs/overview). RPC (remote procedure call) the idea is that we can call a method on a server as we were calling a local object.


So the first thing I did was visit the official site for [Grpc](https://grpc.io) and went straight to the ruby section. What I found is a tutorial that is not up to date with the code from the [repo](https://github.com/grpc/grpc) at Github, and I found a little hard to follow, so I decided to merely extract the tutorial to my repo and explaining the overview of what I learned.


Please, I want to make clear that the majority of the code is identical to the one in the grpc repo but I organized and rename a couple of things, so is easier to understand.

The first piece in our puzzle is the definition of our Grpc service, for that we use a file with `proto` extension.

If you never use protocol buffers before do not worry it may sound scary, but it is a pretty simple idea.
Inside our proto file, we define the different types that we use both for the client and the server, you can think of them as objects.

```
message Coordinate {
  int32 latitude = 1;
  int32 longitude = 2;
}
```

You can even use already defined types inside other types.

```
message Area {
  Coordinate lo = 1;
  Coordinate hi = 2;
}
```

There are are many more features in [protocol buffers](https://developers.google.com/protocol-buffers/docs/proto3), but I am not going cover them here.

Now that we have our basic types defined we are going to create our service; you can think of it as our API on our server.

We need to define a service and declare the different methods that the clients can call.

```
service RouteGuide { # this will the name of our service
}
```

Inside our service, we define the different `rpc` calls.


There are 4 ways a client can communicate with the server:

The client sends a request, and the server sends a response, this is the simplest one.

`rpc GetLocation(Coordinate) returns (Location) {}`


The client sends a request, and the server sends a stream of messages back, then the client reads them.

`rpc ListLocations(Area) returns (stream Location) {}`


The client sends a stream of messages then the client waits for the server to read them all and send a response.

`rpc RecordRoute(stream Coordinate) returns (RouteSummary) {}`


Last option and more complicated one, bidirectional were both sides send a read-write stream, the two stream works independently so clients and servers can read and write in whatever order they like. The order of messages in each stream is preserved.

`rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}`

That is all that we need to know. Here is the complete [file](https://github.com/GustavoCaso/grpc_ruby_demo/blob/master/protos/route_guide.proto)

Now we are going to start working on the Ruby implementation of the server and the client.

We need these dependencies to read our `proto` file and transform into something our ruby code could understand

```ruby
gem install "grpc"
gem install "grpc-tools"
```

Thanks to `grpc-tools` we can use a command to transform our `proto` file into a `ruby` file.


```
bundle exec grpc_tools_ruby_protoc
```
That creates [files](https://github.com/GustavoCaso/grpc_ruby_demo/tree/master/lib/grpc_demo/rpc) with all the objects that we need to work.

### Creating the server

Let's explain what our server does; It has a Hash database of locations where it stores the latitude and longitude as keys, and the value is the name of that location.

- So our server can receive a Coordinate (latitude and longitude) and get a Location back with the information.

- It can receive an Area that is defined between two coordinates and return a list of location within that region.

- It can receive a stream of coordinates an calculate the route summary between this coordinates.

To create the server we need first to create a `Handler` it has the same contract as we previously defined in our proto file, and we described above, so it has corresponding methods for all the `rpc` calls we have previously defined

Thanks to the generated files we have access to some classes that help to define our `Handler`.
So we are going to start by creating a class that extends from `RouteGuide::Service` class generated from the [proto](https://github.com/GustavoCaso/grpc_ruby_demo/tree/master/lib/grpc_demo/rpc) file.

Let's implement the `GetLocation` contract.

```ruby
def get_location(coordinates, _call)
  name = DB.find(longitude: coordinates.longitude, latitude: coordinates.latitude) || ''
  Location.new(coordinates: coordinates, name: name)
end
```

The `DB` is the `Hash` database that we previously mentioned. So if we read the method it is quite clear what is doing, is receiving a coordinate as an argument, is looking inside the `DB` for the name of that coordinate and is returning a new `Location`. Remember that `Location` is something that we generated from the `proto` file.

Now, let's try with the server stream example.
The contract said that the server would receive a rectangle and it streams points back to the client.
To achieve this, we need to return an [Enumerator](https://ruby-doc.org//core-2.2.0/Enumerator.html) that yields our points to the client

```ruby
def list_locations(area, _call)
  Locations.new(area).each
end
```

The logic for yielding points is encapsulated inside the `Locations` you can see the full code [here](https://github.com/GustavoCaso/grpc_ruby_demo/blob/master/lib/grpc_demo/server/locations.rb)

Let's see the example when the client sends a stream of points to the server. When the server receives a stream, it gets an [Enumerator](https://ruby-doc.org//core-2.2.0/Enumerator.html) where it reads all the data.

```ruby
def record_route(call)
  RecordRoute.new(call).call
end
```

On the `call` object from the arguments, we can use the method `each_remote_yield` that yield each message sent from the client.

```ruby
call.each_remote_read do |point|
  # logic inside
end
```

You can find the full code [here](https://github.com/GustavoCaso/grpc_ruby_demo/blob/master/lib/grpc_demo/server/record_route.rb).

Here is all the code for the [Handler](https://github.com/GustavoCaso/grpc_ruby_demo/blob/master/lib/grpc_demo/server/handler.rb) class.

#### Starting the server

That is the painless part once we have all the logic for the Handler, we just need to create a new instance of `GRPC::RpcServer`, and we have our server running waiting for clients to send data to it.

```ruby
server = GRPC::RpcServer.new
server.add_http2_port("0.0.0.0:50051", :this_port_is_insecure)
server.handle(Handler.new)
server.run_till_terminated
```

### Creating the client

When dealing with client code, we need to create what is called a `stub` it has all the method that we have defined in our service `route_guide` such as `get_location`, `list_locations`, `record_route`, basically is the way we communicate with the server.

```ruby
stub = Routeguide::RouteGuide::Stub.new("localhost:50051", :this_channel_is_insecure)
```

Let's communicate with the server to get a feature based on a coordinate; we need to send a `Coordinate` to the server.

```ruby
point = Coordinate.new(latitude:  409_146_138, longitude: -746_188_906)
```

With that, we can call `get_location` passing the coordinate and expect to have a response from the server.

```ruby
response = stub.get_location(coordinate)
response.name # => 'Berkshire Valley Management Area Trail, Jefferson, NJ, USA'
response.coordinates # => <Routeguide::Coordinate: latitude: 409146138, longitude: -746188906>
```

Let's look at an example where the server returns as a stream of data. The server returns an `Enumerator`, and we can loop over it reading the multiple responses, it feels like executing a `method` from a local object.

```ruby
rectangle = Area.new(
  lo: Coordinate.new(latitude: 400_000_000, longitude: -750_000_000),
  hi: Coordinate.new(latitude: 420_000_000, longitude: -730_000_000))

responses = stub.list_locations(area)
responses.each do |r|
  puts "- found '#{r.name}' at #{r.coordinates.inspect}"
end
```

Lastly, let's look an example when the client sends a stream of data to the server; it works similarly as the server implementation it should send an `Enumerator` which yield each message to the server.

```ruby
class RandomRoute
  attr_reader :size

  def initialize(size)
    @size = size
  end

  def each
    return enum_for(:each) unless block_given?
    size.times do
      feature = DB.rand
      point = create_point(feature[:location])
      yield point
    end
  end

  private

  def create_point(location)
    Routeguide::Coordinate.new(Hash[location.each_pair.map { |k, v| [k.to_sym, v] }])
  end
end

points_on_route = 10  # arbitrary
request = RandomRoute.new(points_on_route)
response = stub.record_route(request.each)
puts "summary: #{response.inspect}"
```

Our class `RandomRoute` encapsulate the logic for creating an `Enumerator` using the Kernel method `enum_for` there is an excellent [article](https://blog.arkency.com/2014/01/ruby-to-enum-for-enumerator/) that explain it more in depth.

With all of this, we just created a Grpc Server and a Client that communicate with each other.

I have created a [repo](https://github.com/GustavoCaso/grpc_ruby_demo) with all the code so you can have a look.

If you have any thoughts or questions, please share, and I will be happy to answer in the comments.

Thank you for reading I know it has been a long journey, but I hope you have learned something new today.
