---
layout: post
title: "React back to basics"
date: 2016-09-18 06:57:40 +0200
comments: true
categories: [React, Javascript]
---

## React Components

This is not going to be another post describing [React](https://facebook.github.io/react/) and what is good or what is bad about it, I'm just learning it, and part of the learning process I decided that I will write a blog post for helping me maintain this new concepts. I will avoid many concepts, like how to create a project for importing all the libraries, I will create another post regarding that topic.

#### Component

The basic idea behind a component, is that is a reusable piece of code, every component has to have a `render` function that returns the actual component, in this examples I'll be using `JSX`, so if you want to check it [JSX in Depth](https://facebook.github.io/react/docs/jsx-in-depth.html), this basically lets us write some sort of `html` inside our JavaScript file, that when compiled it will look more Reactable.

```js
var Nav;
// Input (JSX):
var app = <Nav color="blue" />;
// Output (JS):
var app = React.createElement(Nav, {color:"blue"});
```

Back to our `Component` lets start with a simple one:

```js
class Example extends React.Component {
  render(){
    return(
      <div>
        <p> Hello </p>
      </div>
    )
  }
}
```

So this is our initial component, to make use of it, we will have to tell React to render it in the DOM.

```js
ReactDOM.render(<Example/>, document.getElementById('app'))
```
<!-- more -->

So now our component has been render inside our `<div id="app"></div>`.

But this isn't really useful, we could achieve the same thing with plain html, wait there is more to it.

##### Props

Every component has it's own props, that can be access from inside the component, and that we can pass it from outside the component, just like arguments to a function. To keep the example clear I will just pass the text that I want to display inside.

```js
ReactDOM.render(<Example text="Hello from the props"/>, document.getElementById('app'))
```

To access it's `props` we can use `this.props` inside our component.

```js
class Example extends React.Component {
  render(){
    return(
      <div>
        <p>{this.props.text}</p>
      </div>
    )
  }
}
```

Ok let's recap, we have passed the prop `text` to our component, and to access it we use `this.props.text`, but what with the `{}`. To interpolate it's value inside the render function, we need to surrounded our prop with it.

Now the component can have dynamic content display in the page. But what happen if you forget to include the text props in the component, do not worry we have `defaultProps`.

```js
class Example extends React.Component {
  ...
}

Example.defaultProps = {
  text: 'Using defaults'
}
```

This way React has our back.

##### State

This is another interesting concept, the state is a collection of values manage by our component. Modifying this values will trigger a new render of our component, but I will talk more in-depth about the `render life-cycle` in the next post.

To setup the states values we have to use the `constructor` function.

```js
class Example extends React.Componet {
  constructor(){
    super();
    this.state = { text: 'this is from the state' }
  }

  render(){
    ...
  }
}
```

Now we can use the state with `this.state.text`, but the state without the a way of modifying it is no use, so to do that we have to use `this.setState({})` this will change the state and trigger a new render of the component with the new state.

To get something out of this we are going to build a component that will show the text of the state, but we will be able to changed when typing inside a input. We will make use of [DOM Event Listeners in a Component](https://facebook.github.io/react/tips/dom-event-listeners.html).

So to make this example work, we will have to hook to an event `onChange` that will trigger a function that will update the state.

```js
class Example extends React.Component {
  constructor(){
    super();
    this.state = {
      txt: 'this is the state txt'
    }
  }
  update(e){
    this.setState({txt: e.target.value})
  }
  render(){
    return (
      <div>
        <input type="text"
          onChange={this.update.bind(this)} />
        <h1>{this.state.txt}</h1>
      </div>
    );
  }
}

ReactDOM.render(<Example />,document.getElementById('app'));
```

This simple post is getting quite long for just explaining the basic. So will leave here a life example so you can play with.

<a class="jsbin-embed" href="http://jsbin.com/jevame/embed?html,js,output">JS Bin on jsbin.com</a><script src="http://static.jsbin.com/js/embed.min.js?3.39.18"></script>

In the next post I will get more in the `DOM Events`, the `state`, `render life-cycle` and more concepts that I will be learningthis week.
