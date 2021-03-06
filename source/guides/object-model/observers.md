## 观察器

原文：[http://emberjs.com/guides/object-model/observers/][source_link]

> 译者按：Observer 通常都被译为“观察者模式”，它是一种软件设计模式，有时也叫做“发布/订阅（Pub/Sub）模式”，可参看 [Wikipedia 里的“观察者模式”词条][wiki_observer]。在本指南中之所以译为“观察器”，是因为在这里 Observers 的意思是“Ember.js 框架用来实现观察者设计模式所使用的机制”，因此与惯用译法略有差别。

Ember 为包括计算后属性在内的任意一种属性提供了观察器。你可以通过使用 `addObserver` 方法来为一个对象设置一个观察器。

```javascript
Person = Ember.Object.extend({
  // 这些属性将由 `create` 提供
  firstName: null,
  lastName: null,

  fullName: function() {
    var firstName = this.get('firstName');
    var lastName = this.get('lastName');

    return firstName + ' ' + lastName;
  }.property('firstName', 'lastName'),
  
  fullNameChanged: function() {
    // 处理改变
  }.observes('fullName').on('init')
});

var person = Person.create({
  firstName: "Yehuda",
  lastName: "Katz"
});

person.set('firstName', "Brohuda"); // 观察器将被触发
```

由于 `fullName` 这个计算后属性依赖于 `firstName` 的变化，所以在更新 `firstName` 时也将触发 `fullName` 上的观察器。

### Observers and asynchrony

Observers in Ember are currently synchronous. This means that they will fire
as soon as one of the properties they observe changes. Because of this, it
is easy to introduce bugs where properties are not yet synchronized:

```javascript
Person.reopen({
  lastNameChanged: function() {
    // The observer depends on lastName and so does fullName. Because observers
    // are synchronous, when this function is called the value of fullName is
    // not updated yet so this will log the old value of fullName
    console.log(this.get('fullName'));
  }.observes('lastName')
});
```

This synchronous behaviour can also lead to observers being fired multiple
times when observing multiple properties:

```javascript
Person.reopen({
  partOfNameChanged: function() {
    // Because both firstName and lastName were set, this observer will fire twice.
  }.observes('firstName', 'lastName')
});

person.set('firstName', 'John');
person.set('lastName', 'Smith');
```

To get around these problems, you should make use of `Ember.run.once`. This will
ensure that any processing you need to do only happens once, and happens in the
next run loop once all bindings are synchronized:

```javascript
Person.reopen({
  partOfNameChanged: function() {
    Ember.run.once(this, 'processFullName');
  }.observes('firstName', 'lastName'),

  processFullName: function() {
    // This will only fire once if you set two properties at the same time, and
    // will also happen in the next run loop once all properties are synchronized
    console.log(this.get('fullName'));
  }
});

person.set('firstName', 'John');
person.set('lastName', 'Smith');
```

### Observers and object initialization

Observers never fire until after the initialization of an object is complete.

If you need an observer to fire as part of the initialization process, you
cannot rely on the side effect of set. Instead, specify that the observer
should also run after init by using `.on('init')`:

```javascript
App.Person = Ember.Object.extend({
  init: function() {
    this.set('salutation', "Mr/Ms");
  },

  salutationDidChange: function() {
    // some side effect of salutation changing
  }.observes('salutation').on('init')
});
```

### Unconsumed Computed Properties Do Not Trigger Observers

If you never `get` a computed property, its observers will not fire even if
its dependent keys change. You can think of the value changing from one unknown
value to another.

This doesn't usually affect application code because computed properties are
almost always observed at the same time as they are fetched. For example, you get
the value of a computed property, put it in DOM (or draw it with D3), and then
observe it so you can update the DOM once the property changes.

If you need to observe a computed property but aren't currently retrieving it,
just get it in your init method.


### Without prototype extensions

在没有原型扩展的前提下使用 Ember 的时候，你可以用 `Ember.observer` 方法来定义内联式观察器

```javascript
Person.reopen({
  fullNameChanged: Ember.observer(function() {
    // 这是内联式版本的 .addObserver
  }, 'fullName')
});
```

### Outside of class definitions

You can also add observers to an object outside of a class definition
using addObserver:

```javascript
person.addObserver('fullName', function() {
  // deal with the change
});
```
