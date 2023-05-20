+++
title = 'Working your way up with promises'
date = 2015-10-08
tags = ["JavaScript"]
+++

Lately, I have been working more with `JavaScript`.

Initially, I wasn't enthusiastic about working with `JavaScript,` but the project surprised me.

For starters, it uses [React](https://facebook.github.io/react/) and [Redux](http://rackt.github.io/redux/) for managing the state of the application.

It has been a significant change to understand how they work and wrapping my head around the concept of [Reducers](http://rackt.github.io/redux/docs/basics/Reducers.html), [Store](http://rackt.github.io/redux/docs/basics/Store.html) and [Actions](http://rackt.github.io/redux/docs/basics/Actions.html)
But it is fantastic working with these new technologies.

But what surprised the most was the new API for the [Promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise).

For me, it changed the way for working with asynchronous actions.

A promise is an operation that has yet to be completed but is expected in the future.
Working with a promise, you must pass an executor or function and two callbacks as their arguments.

``` js
var promise = new Promise(function(callback1, callback2){
  executor body
})
```

For better understanding, we will call the callbacks `resolve` and `reject`; those will be the actions that will take place after the body of the function or `executor.`
finish.

Promises bring us a few methods that help us interact with them, like `then`, `catch`, `all`, and a few more; I'm not an expert, but I recommend looking into the documentation.
But for the sake of the post, it is like an Ajax call with all the callbacks.

With the previous example, I will show a simple way of fetching a random joke from the `Chuck Norris Database`.

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

The example is pretty simple we make an asynchronous call to get a random joke, and on the `onload` function, if the response is correct,
we execute the resolve callback. If not, we perform the reject callback.

The `then` will wait for the get request to finish and execute the callback.

With this new Promise API, you can do impressive things.

One of the things that I requested on my job was the possibility of fetching some data from the server, but the server was executing a background job, so there was no way of knowing when the job had finished.
In this scenario, we could use `Promises` to fetch data from the API, doing polling until the server returns an answer.

This example is more tricky.

The main idea is to halt the execution until we get an answer from the server; you could display a spinner, a loading message, whatever you want.

OK, down to business.

``` js
function getData() {
  var interval;
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

We call a function that returns a promise to fetch the server's data.
It will try to get the data every 10 seconds; when the data is collected, we will call the `resolve` function to finish the promise and clear the interval.

This code `getData().then(displayData)` will be called, but the `resolve` function will be executed only when the data is fetched from the server.
