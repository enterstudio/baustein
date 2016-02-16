# baustein

[![Build Status:](https://secure.travis-ci.org/ssaw/baustein.png?branch=master)](https://travis-ci.org/ssaw/baustein)

Baustein is a component library that we use at [Lyst](http://www.lyst.com) to build the front-end of our website. It has some similarities with other component-based frameworks such as [React](https://facebook.github.io/react/) or [FruitMachine](https://github.com/ftlabs/fruitmachine), but grew very organically out of the specific needs we had at [Lyst](http://www.lyst.com).

## Docs
* [The basics](#the-basics)
* [Handling DOM events](#handling-dom-events)
* [Using options](#using-options)
* [Creating components programmatically](#creating-components-programmatically)
* [Working with templates](#working-with-templates)
* The render context
* Handling custom events


## The Basics
The simplest use case for Baustein is to progressively enhance some server-rendered HTML, so let's look at how we can do this.

Imagine we have a webpage with the following server-rendered HTML.

```html
<div is="my-component">Hello!</div>
```

This HTML by itself will do nothing, even with Baustein loaded on the page. For anything to happen we need to do two things, register a component with the name **my-component** (denoted by the `is` attribute), and then initialise Baustein.

Baustein keeps an internal registry of available components and to add a component to this registry you use `baustein.register(name, implementation)`.

```js
import baustein from 'baustein';

// register our component
baustein.register('my-component', {

    init() {
        console.log('Hello from my-component!');
    }

});

// initialise baustein
baustein.init();
```

The call to `baustein.init()` will cause Baustein to parse the DOM looking for any components, initialising them as it finds them, and then to listen for any changes to the DOM so it can check to see if new components have been added. `baustein.init()` should only be called once all your components are registered, and should only be called once.

This example shows how to register a component and to initialise Baustein, but logging things to the console isn't very interesting. Let's look at how we can respond to DOM events.


## Handling DOM Events
To show how to listen to and respond to DOM events let's create a counter button component that shows how many times it's been clicked.

First we'll start with some HTML:

```html
<button is="counter-button">Clicked <span>0</span> times</button>
```

Now lets register our component.

```js
baustein.register('counter-button', {

    init() {
        this.count = 0;
    },

    setupEvents(add) {
        add('click', this.onClick);
    },
    
    onClick(event) {
        this.count++;
        this.el.querySelector('span').textContent = this.count;
    }

});
```

This component uses two Baustein API methods, `init` and `setupEvents`. `init` is called when a component is created and is a good place to initialise any instance properties. `setupEvents` is also called when a component is created and is passed a function that can be used to bind events. This function accepts either two or three arguments:

```js
setupEvents(add) {

    // this will listen for clicks anywhere within the component
    add('click', this.onClick);
    
    // this will only listen for clicks that happen within the <span> element inside the component
    add('click', 'span', this.onSpanClick);
}
```

In this above example `onClick` will receive just the `event` object as it's only argument, but `onSpanClick` will receive the `event` object as it's first argument **and** the `<span>` element as it's second argument. 

> Note that the handler function does not need to be bound with `.bind(this)` or anything similar as baustein will always call the function with the instance of the component as the `this` value.

To illustrate event handling further let's create a slightly more complicated component:

```html
<ul is="highlightable-list">
    <li>John</li>
    <li>Paul</li>
    <li>George</li>
    <li>Ringo</li>
</li>
```

```js
baustein.register('highlightable-list', {

    setupEvents(add) {
        add('click', 'li', this.onItemClick);
    },
    
    onItemClick(event, clickedItem) {
        const items = Array.from(this.el.querySelectorAll('li'));
        items.forEach(item => item.classList.toggle('active', item === clickedItem));
    }

});
```

This component listens for click events on any `<li>` element within it, and when it receives one it adds a class of `active` to that `<li>` and removes it from any others.

### Global Events
Global events are ones that happen outside of a component, for example `resize` and `scroll` events, which typically happen on the `window`. Listening for these events is done in a slightly different way, using the `setGlobalHandler(event, handler)` method. A good place to start listening for these kinds of events is `onInsert` which is called whenever a component is inserted into the DOM.

```js
onInsert() {
    this.setGlobalHandler('resize', this.onResize);
},

onRemove() {
    this.releaseGlobalHandler('resize', this.onResize);
},

onResize() {
    this.el.style.width = `${window.innerWidth}px`;
}
```

Due to the performance implications of dispatching high frequency events like `scroll` to components it is important that a component only listens for these events when it really needs to. In the above example we "release" the event handler when the component is removed using the `onRemove` function, which is called whenever a component is removed from the DOM.


## Using Options
Components can take options which are passed using attributes. To show how this works lets extend our previous `highlightable-list` component to take the "active" class as an option.

```html
<ul is="highlightable-list" active-class="highlighted">
    <li>John</li>
    <li>Paul</li>
    <li>George</li>
    <li>Ringo</li>
</li>
```

```js
baustein.register('highlightable-list', {

    setupEvents(add) {
        add('click', 'li', this.onItemClick);
    },
    
    onItemClick(event, clickedItem) {
        const items = Array.from(this.el.querySelectorAll('li'));
        const {activeClass} = this.options;
        items.forEach(item => item.classList.toggle(activeClass, item === clickedItem));
    }

});
```

As you can see we can extract the options from `this.options`, which will be an object where the keys are camel-cased versions of the attribute names and the values are the attribute values. The values are run through `JSON.parse()` which means that they can be numbers, booleans, lists, or even objects. If parsing as JSON fails then the value will be left as-is.

### Providing Default Options
It is possible to provide default options by implementing `defaultOptions` on the component. If provided then this must be either an object, or a function that returns an object. To extend our `highlightable-list` component to have a default value for `activeClass` we can do:

```js
baustein.register('highlightable-list', {

    defaultOptions: {
        activeClass: 'active'
    },

    setupEvents(add) {
        add('click', 'li', this.onItemClick);
    },
    
    onItemClick(event, clickedItem) {
        const items = Array.from(this.el.querySelectorAll('li'));
        const {activeClass} = this.options;
        items.forEach(item => item.classList.toggle(activeClass, item === clickedItem));
    }

});
```

## Creating components programmatically
Although progressively enhancing server-rendered HTML is a big use case of Baustein it is also possible to create components programmatically. The `baustein.register()` function returns a constructor function which can be used to create a new instance of the component.

```js
const HighlightableList = baustein.register('highlightable-list', {
    // implementation omitted
});

// create a new instance of the component
const list = new HighlightableList({
    activeClass: 'highlighted'
});

// append it to the <body>
list.appendTo(document.body);
```

The above code creates a new instance of the component, passing options via an object to the constructor, and then appends the component to the `<body>`. However if you inspect the `<body>` you would see something like this:

```html
<div is="highlightable-list" data-component-id="1"></div>
```

This is obviously nothing like the HTML we were using before for this component, but so far the component doesn't know anything about how to render itself. Let's change that.

Components can implement a `template` function, which should return an HTML string. Implementing the render function for `highlightable-list` might look like this:

```js
baustein.register('highlightable-list', {

    tagName: 'ul',

    template() {
        return `
            <ul is="highlightable-list">
                ${this.options.items.map(item => `<li>${item}</li>`).join('')}
            </ul>
        `;
    },

    defaultOptions() {
        return {
            activeClass: 'active',
            items: []
        };
    },

    setupEvents(add) {
        add('click', 'li', this.onItemClick);
    },
    
    onItemClick(event, clickedItem) {
        const items = Array.from(this.el.querySelectorAll('li'));
        const {activeClass} = this.options;
        items.forEach(item => item.classList.toggle(activeClass, item === clickedItem));
    }

});
```

We did three things here, first we added a property called `tagName` which tells Baustein what type of element to create as the root node of the component. Then we implemented the `template` function, which builds and returns some HTML, and finally we changed `defaultOptions` to be a function as now it contain an `items` property which defaults to an empty list. The reason we need to use a function now is that Baustein doesn't deep copy the options from `defaultOptions`, which means in this case if we implemented `defaultOptions` as a property then all component instances would share the same default value for `items`.

Now we can create our component passing some items:

```js
// create a new instance of the component
const list = new HightlightableList({
    items: ['John', 'Paul', 'George', 'Ringo']
});

// append it to the <body>
list.appendTo(document.body);
```

Now if we inspect the body we should see something like our original HTML.

```html
<ul is="highlightable-list" data-component-id="2">
    <li>John</li>
    <li>Paul</li>
    <li>George</li>
    <li>Ringo</li>
</li>
```

## Working with templates

Obviously building HTML strings manually in functions is not a very scalable way to write HTML, but since all Baustein requires is a function that returns an HTML string you are free to use whatever templating language you want. At Lyst we use [Jinja2](http://jinja.pocoo.org/docs/dev/) templates in our Django backend and then use [jinja-to-js](https://github.com/jonbretman/jinja-to-js) to compile these templates into JS functions, meaning we can share the same templates been client and server.
