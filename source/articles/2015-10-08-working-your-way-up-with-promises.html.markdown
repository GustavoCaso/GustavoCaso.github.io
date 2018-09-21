---
layout: post
title: "Working your way up with promises"
date: 2015-10-08 23:06:35 +0200
comments: true
tags: JavaScript
---

#####Promises, what ??

I consider myself a ruby developer, not one with long time professional experience, but one with a great attitude and eager to learn new technologies.

Lately I have been working more with `JavaScript` at the beginning I wasn't really enthusiastic about the idea of working with `JavaScript` but the project itself really surprise me.

READMORE

For starters it was built with [React](https://facebook.github.io/react/) and [Redux](http://rackt.github.io/redux/) for managing the state of the application.
It has been a big change to understand how they work and wrapping my head around the concept of [Reducers](http://rackt.github.io/redux/docs/basics/Reducers.html), [Store](http://rackt.github.io/redux/docs/basics/Store.html) and [Actions](http://rackt.github.io/redux/docs/basics/Actions.html)
but I could say is awesome working with this new technologies.

But what really shine the most was the new API for the [Promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise), for me change the way for working with asynchronous actions.


I promise is an operation that hasn't complete yet, but is expected in the future.
To work with a promise you will have to pass an executor or function and two callbacks as their arguments.

``` js
  var promise = new Promise(function(callback1, callback2){
    executor body
  })
```

For better understanding we will call the callbacks `resolve` and `reject`, those will be the actions that will take place after the body of the function or `executor`
finish.

Promises bring us a few methods than help us interact with them like `then`, `catch`, `all` and a few more, I'm not an expert, but I really recommend to have a look into the documentation.
But for the sake of the post I will say is like an ajax call with all the callback.

With the previous example I will show a simple way of fetching a random joke from the `Chuck Norris Database`.

``` js
  var promise = new Promise( function(resolve, reject) {
    var client = new XMLHttpRequest();
    client.open('GET', 'http://api.icndb.com/jokes/random')

    client.onload = function(){
      if (client.status == 200){
          resolve(client.response);
        } else {
          reject(Error(client.statusText));
        }
    };
    client.send()
  });

  promise.then(function(data){
    console.log(data)
  }, function(statusText){
    console.log(statusText)
  })
```

The example is pretty simple we make an asynchronous call to get a random joke and on the `onload` function if the response is correct
we execute the resolve callback if not we execute the reject callback.

The `then` will wait for the get request to finish and then will execute the callback.

There is so much power with this new `Promise API` you can actually do really impressive things.

One of the things that request on my job was the possibility to fetch some data from the server, but the server was executing a background job, so there is no way of knowing when the job has finished.
In this scenario we could use `Promises` to fetch data from the api doing some kind of polling until the server return an answer.

This is example is a little more tricky, but nothing you guys can understand.
So the main idea is to halt the execution until we get an answer from the server, you could display a spinner a loading message whatever you want.

OK down to business.

``` js
function getData() {
  var intervalId;
  var promise = new Promise(function(resolve, reject){
    function makeConnection(){
      var client = new XMLHttpRequest()
      client.send('Get', 'http://fake_endpont.com/pai/1/info.json')

      client.onload = function() {
        if (client.status === 200 && client.response.message === 'complete'){
          clearInterval(intervalId)
          resolve(client.response)
        }
      }
    }


    intervalId = setInterval(makeConnection, 1000)
  });

  return promise;
}

function displayData(response) {
  console.log(response);
}


getData().then(displayData)

```

Thanks to a colleague at work who help me out I was able to do it and understand it, basically we call a function that returns a promise, which executor will call a function that will fetch the data from the server,
 and will try to get the data every 10 seconds, when the data is what we want will call the resolve function finishing the promise and clear the interval.

This code `getData().then(displayData)` will be call, but the `resolve` function will be executed when the code fulfill the condition.

Is has been a long post, I thanks for reading and any mistake please don't hesitate and comment.

