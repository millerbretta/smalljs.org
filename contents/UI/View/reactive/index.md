---
title: Reactive
date: 2014-03-11
summary:
  github: component/reactive
  where:
    npm: reactive
    Component: component/reactive
    Bower: N/A
  supported_browsers: IE9+
  file_size:
    loc: 599
    minified: 
      self: 9.5k
      standalone: 19.6k
  dependencies:
    classes: https://github.com/component/classes
    event: https://github.com/component/event
    query: https://github.com/component/query
    emitter: https://github.com/component/emitter
    domify: https://github.com/component/domify
    carry: https://github.com/yields/carry
    debug: https://github.com/visionmedia/debug

---
Knockout and AngularJS have popularized the concept of declarative data binding in UI views. [Reactive](https://github.com/component/reactive) is a library that gives you this convinience in à la carte fashion.

*Note: reactive does 1-way data binding: from the model to the view - it does not support 2-way binding. It can be made to approach 2-way binding behavior by declaring custom bindings.*

## Basic Example

Supposed you have the following "view" in your HTML page, which displays information about a person. It uses string interpolation to substitute in the `name` and `age` properties of said person:

``` html
<div id="person-view">
  <label>Name:</label>
  <span>{ name }</span>
  <label>Age:</label>
  <span>{ age }</span>
</div>
```

You have a person object:

``` js
var bob = {
  name: 'Bob',
  age: 38
};
```

To wire up the view to the object:

``` js
var elm = document.getElementById('person-view');
reactive(elm, bob);
```

Bam! Now the view is populated with the values from the object.

Alternatively, the first argument can also be a string containing the template markup.

``` js
reactive('<div id="person-view">...</div>', bob);
```

But, since `bob` is a plain object, there is no way to tell the view that its value has updated. What we need is something that fires events - we can use an [event emitter](/object/events/event-emitter/) for this.

## Event Emitter

An `Person` object that emits events can be written as:

``` js
function Person(name, age){
  this.name = name;
  this.age = age;
}
Person.prototype = new EventEmitter; // or substitute your preferred
                                     // method of inheritance here.
```

`bob` can be instantiated as

``` js
var bob = new Person('Bob', 38);
```

And now, it can emit events to tell the view when its properties has changed:

``` js
bob.age = 39;
bob.emit('change age');
```

At this point the age value in the view's display will update. Note that the name of the event reactive looks for by default is `change <name of property>`.

## Backbone Model

If you use Backbone, I know what you are saying: *why trigger the event manually? Backbone models will do that for you!* Valid point. Reactive can do that! Reactive is pluggable and can be made to work with a variety of model implementations, including Backbone models. To do this, we need an adapter for backbone models. A model adapter is a constructor that takes a model object and implements the following methods:

* subscribe
* unsubscribe
* unsubscribeAll
* set
* get

See the [adapters documentation](https://github.com/component/reactive#adapters) for more details on how to implement your own adapter. I've made it easy for you, here's the code for the Backbone adapter:

``` js
function BackboneAdapter(model){
  if (!(this instanceof BackboneAdapter)){
    return new BackboneAdapter(model);
  }
  this.model = model;
}

BackboneAdapter.prototype = {
  subscribe: function(prop, fn){
    // This tells it how to bind a property change event
    this.model.on('change:' + prop, fn);
  },
  unsubscribe: function(prop, fn){
    // This tells it how to unbind a property change event
    this.model.off('change:' + prop, fn);
  },
  unsubscribeAll: function(){
    // This tells it how to unbind all events
    this.model.off();
  },
  set: function(prop, value){
    // This tells it how to set a property on a model
    this.model.set(prop, value);
  },
  get: function(prop, value){
    // This tells it how to get a property on a model
    return this.model.get(prop);
  }
}
```

*Note: I've put the Backbone adapter on github as a [separate project](https://github.com/airportyh/reactive-backbone).*

Now, to test it out, first make a backbone model instance:

``` js
var bob = new Backbone.Model({name: 'Bob', age: 35});
```

Bind the backbone model to the view using reactive, supplying the backbone adapter

``` js
reactive(elm, bob, {adapter: BackboneAdapter});
```

And voila! Now the view will automatically update when the model's attributes are set - such as `bob.set('name', 'Robert')`. 

Everyone loves Backbone, right? If Backbone is not your thing though, that's okay too! For alternative model implementations, have a look at [Bamboo](https://github.com/defunctzombie/bamboo) and [Modella](https://github.com/modella/modella).

## String Interpolation

Now you are ready to take a closer look at Reactive's the string interpolation feature. Take this example:

``` html
<article>
  <h2>{ name }</h2>
</article>
```

Reactive interpolates expressions between `{` and `}`. The syntax can be simple properties, but can also be more complex JavaScript expressions, such as method calls:

```
{ name.toUpperCase() }
```

and string concatenation:

```
{ firstName + ' ' + lastName }
```

The properties used in these expressions will be bound automatically so that if they change, the view updates correctly.

## Declarative Bindings

In addition to string interpolation, Reactive also provides declarative bindings - which are written as attributes of DOM elements. I'll walk through the most useful ones.

### Event Bindings

You can use an event binding to delegate event handling to your code. To specify a handler for a `click` event on an element, you'd use the `on-click` binding:

``` html
<button on-click="save">Save</button>
```

The value specified - in this case: `save` - maps to a method in the delegate object, which you'll need to specify as an option to reactive. For example:

``` js
var delegate = {
  save: function(e, view){
    var model = view.model;
    request.post('/api/person/', model.serialize());
  }
}

reactive(elm, bob, {
  delegate: delegate
});
```

All the common DOM events are supported, via `on-<event name>`.

### `each` Binding

`each` gives you the ability to iterate an array of objects. It binds a model property - the value of which should be an array - and renders the original contents of the element once for each item in the array. Example:

``` js
<ul each="children">
  <li>{last}, {first}</li>
</ul>
```

*Note: in order to be notified of changes in the array, the `each` binding [duck punches](http://www.paulirish.com/2010/duck-punching-with-jquery/) the instance of array that's in use.*

If you use Backbone, you might be wondering whether this can iterate a `Backbone.Collection`. Currently the answer is no.

### `data-<attr>` Binding

`data-<attr>` binds a model property to an attribute of the element. 

``` html
<a data-href="download_url">Download</a>
```

### `data-visible` and `data-hidden` Bindings

`data-visible` binds a boolean model property. If the value is true, it adds the class `visible` to the element, otherwise, it addes the class `hidden` to the element. This allows for conviniently showing or hiding it - note that you'll need to write CSS rules to instruct the browser how you want to show or hide the element based on these classes (you could use CSS animations or simply display: none). `data-hidden` does the opposite of `data-visible`.

``` html
<p data-hidden="hasItems">no items</p>
<ul data-visible="hasItems">
  <li each="items">{name}</li>
</ul>
```

### `data-checked` Binding

`data-checked` binds a model property to the `checked` property of a checkbox.

``` html
<input type="checkbox" data-checked="agreed_to_terms">
```

### `data-append` Binding

`data-append` appends a new DOM element.

``` html
<div data-append="histogram"></div>
```

The `histogram` property of the model is expected to contain a DOM element containing the desired widget.

### More On Bindings

[Read the docs](https://github.com/component/reactive#declarative-bindings) for refenences on all the bindings. It is also possible to write your own [custom bindings](https://github.com/component/reactive#writing-bindings) that do amazing things! I'll leave that as an exercise for the reader.

## A View Pattern

There is a view pattern that works well for applications that use Reactive. It looks like this

``` js
function UserView(user){
  this.user = user;
  this.view = reactive(html, user, {
    delegate: this // event delegate is set to the instance itself
  });
}

// Event handlers via event binding can simply be written as methods
UserView.prototype.edit = function(evt){
  ...
}
```
