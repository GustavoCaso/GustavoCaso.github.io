+++
title = 'React back to basics'
date = 2016-09-18
tags = ["React", "JavaScript"]
+++

This post will not be another post describing [React](https://facebook.github.io/react/) and what is good or wrong about it; I'm just learning it, and part of the learning process I decided to write a blog post to help me maintain this new concept.

#### Component

The basic idea behind a component is that it is a reusable piece of code; every component has to have a `render` function that returns the actual HTML.

I'll be using `JSX,` so if you want to check it [JSX in Depth](https://facebook.github.io/react/docs/jsx-in-depth.html)

```js
var Nav;
// Input (JSX):
var app = <Nav color="blue" />;
// Output (JS):
var app = React.createElement(Nav, {color:"blue"});
```

Back to our `Component,` let's start with a simple one:

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

So this is our initial component; to use it, we must tell React to render it in the DOM.

```js
ReactDOM.render(<Example/>, document.getElementById('app'))
```

So now our component has been rendered inside our `<div id="app"></div>.`

But this could be more useful; we could achieve the same thing with plain HTML; wait, there is more to it.

##### Props

Every component has its props that can be accessed from inside the component and that we can pass from outside the component, just like arguments to a function. To keep the example clear, I will pass the text that I want to display inside.

```js
ReactDOM.render(<Example text="Hello from the props"/>, document.getElementById('app'))
```

To access its `props,` we can use `this.props` inside our component.

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

Let's recap: we have passed the prop `text` to our component, and to access it, we use `this.props.text`, but what with the `{}`? We need to surround our prop with it to interpolate its value inside the render function.

Now the component can have dynamic content displayed on the page. But what happens if you need to remember to include the text props in the component, do not worry. We have `defaultProps.`

```js
class Example extends React.Component {
  ...
}

Example.defaultProps = {
  text: 'Using defaults'
}
```

This way, React has our back.

##### State

The state is another exciting concept; the state is a collection of values managed by our component. Modifying these values will trigger a new render of our component, but I will talk more in-depth about the `render life-cycle in the next post.

We have to use the `constructor` function to set up the state values.

```js
class Example extends React.Component {
  constructor(){
    super();
    this.state = { text: 'this is from the state' }
  }

  render(){
    ...
  }
}
```

Now we can use the state with `this.state.text`, but the state without a way of modifying it is no use, so to do that, we have to use `this.setState({})`. Calling `setSate` will change the state and trigger a new render of the component with the new state.

To get something out of this, we will build a component that will show the text of the state, but we will be able to change it when typing inside an input. We will use [DOM Event Listeners in a Component](https://facebook.github.io/react/tips/dom-event-listeners.html).

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

This simple post is getting quite long for explaining the basic. So leave here a living example that you can play with.

<a class="jsbin-embed" href="http://jsbin.com/jevame/embed?html,js,output">JS Bin on jsbin.com</a><script src="http://static.jsbin.com/js/embed.min.js?3.39.18"></script>

In the next post, I will get more into the `DOM Events,` the `state,` `render life-cycle and more concepts I will be learning this week.
