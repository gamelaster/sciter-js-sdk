
Note: this chapter is modelled after [RectJS/State and Lifecycle](https://reactjs.org/docs/state-and-lifecycle.html) article. It is conceptionally close but **uses different terminology**.

## Data [Model] and Component Lifecycle

Sciter's Reactor, in general, is all about components that employ Model-View-Controller idiom.

You have some *data* to be presented to the user. User is able to *view* the data and manipulate it. Manipulation of data is a set of code functions and event handlers that, as a group, are named as *Controller*.

### Component Class

Let's transform our [ticking clock sample](https://sciter.com/docs/content/reactor/rendering.htm##updating-the-rendered-element) into reusable Clock component.

Each Reactor class based component shall have `render()` method:

```JavaScript
class Clock extends Element {
  function render() {
    return <div>
        <h1>Hello, world!</h1>
        <h2>It is {this.time.toLocaleTimeString()}.</h2>
      </div>;
  }
}
```

The `render()` method will be called each time an update happens, but as long as we render `<Clock />` into the same DOM node, only a single instance of the `Clock` class will be used. This lets us use additional features such as local state and lifecycle methods.

## Adding Local Data Storage to a Class

By convention local data storage of a component is located in instance's `this.data` field:

```JavaScript
class Clock extends Element 
{
  // declaring local data storage with initial value 
  time = new Date(); 

  function render() {
    return <div>
        <h1>Hello, world!</h1>
        <h2>It is {this.time.toLocaleTimeString()} now.</h2>
      </div>;
  } 
} 
```

and our `render()` method renders now content of `this` properties.

Class Clock, in its current version, just renders static time determined at the time of its creation. Let's transform it to a running clock.

## Adding Lifecycle Methods to a Class

We want to set up a timer whenever the `Clock` is rendered to the DOM for the first time - when real DOM element is created. In ReactJS this is called “mounting”. Sciter runtime will call `componentDidMount()` method when instance of a class is attached to the DOM element:

```JavaScript
class Clock extends Element 
{
  time = new Date(); // setting initial state 

  function componentDidMount() { // instance of the class is attached to real DOM
    this.timer(1000, () => {
      this.componentUpdate({ time:new Date() });
      return true; // to keep the timer ticking
    });
  }

  function render() {
    return <div>
      <h1>Hello, world!</h1>
      <h2>It is {this.time.toLocaleTimeString()} now.</h2>
     </div>;
  } 
} 
```

You see here new entity - implementation of the componentDidMount() method that starts ticking timer.

The timer callback function is where the magic happens:

```JavaScript
() => {
  this.componentUpdate({ time:new Date() });
  return true; // to keep the timer ticking
}
```

On each timer tick it calls built-in `Element.prototype.componentUpdate(newdata)`method with the new data.

Sciter out of the box provides three lifecycle methods:

* `componentDidMount()` \- called by runtime when the element is attached to the DOM tree and
* `componentWillUnmount()` \- when the element is deleted from the DOM tree. In this particular sample we are not using it if to compare our Clock with the [React's one](https://reactjs.org/docs/state-and-lifecycle.html#adding-lifecycle-methods-to-a-class). `Element.timer()` in Sciter creates timer for concrete DOM element, and as soon as element is destroyed all its timers are freed too.     
* `componentUpdate(props)` - you call this method to update state of the element.

## Updating component state

Built-in class `Element` in Sciter represents DOM element.

Among other methods and properties, the class provides native implementation of `componentUpdate()` method that implements logic close to this:

```JavaScript
class Element
{
  componentUpdate(newdata = null) {
    if(typeof newdata == "object") 
      Object.assign(this,newdata);
    this.post(() => this.patch(this.render()));
  }
}
```

As you see the `componentUpdate(newdata)` does the following:

1. `Object.assign(this,newdata);` - updates variables of this DOM object by values of newdata object;
1. `{ this.patch(this.render()); }` - defines local update method that
   
   * calls `this.render()` method to produces virtual dom element;
   * calls native `Element.patch(velement)` method to patch this existing DOM element using velement definition from the `render()`;
1. `this.post(updater)` - schedules the update invocation to the next event processing cycle.

The delayed rendering logic ensures that multiple `updater()` execution requests are collapsed into single `updater()` invocation request in the event queue.

## Using Date Correctly

There are three things you should know about `componentUpdate(newdata)`.

### Do Not Modify the Data Field Directly

For example, this will not re-render a component:

```
// WRONG!
this.comment = "Hello";
```

Instead, use `componentUpdate()`:

```
// Correct
this.componentUpdate({comment: "Hello"});
```

The only place where you should assign properties directly is the constructor.

### Data Updates are Merged

When you call `componentUpdate(newdata)`, fields of the object you provide are merged into current set of properties of the DOM element.

Therefore your code may issue `componentUpdate()` requests for different parts of the data and they will end up in single reconciliation action between real DOM element and result of `render()` call, so these:

```JavaScript
clock.componentUpdate { time:new Date() };
...
clock.componentUpdate { greeting: "John" };
```

will end up in single `clock.render()` call and screen update.

> Comment to the above. In SciterJS, this:
  ```
  obj.method { name:val };
  ```
  is a short form of a call:
  ```
  obj.method({ name:val });
  ```
  That is just a syntax sugar for making our developer's life a bit easier.

