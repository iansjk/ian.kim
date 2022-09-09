---
layout: "../../layouts/BlogPost.astro"
title: "Checking if a property exists on a JavaScript object"
description: "There are many ways to check if a property exists on an object in JavaScript. I try to explain the difference between them, and which method to use depending on your use case."
pubDate: "Sep 8 2022"
tags: ["js"]
---

Here are the various ways to check if a property exists on an object:

- `myObject.hasOwnProperty(someProperty)`
- `Object.prototype.hasOwnProperty.call(myObject, someProperty)`
- `Object.hasOwn(myObject, someProperty)`
- Accessing it and checking for `undefined` or `null` (`myObject[someProperty] != null` , `myObject.someP != null`)
- `"someProperty" in myObject`

For cases where `myObject` is an object literal that we make ourselves, these methods all return the same thing:

```jsx
const myObject = {
  foo: "bar",
  meaningOfLife: 42,
};

> myObject.hasOwnProperty("foo");
true
> Object.prototype.hasOwnProperty.call(myObject, "foo");
true
> Object.hasOwn(myObject, "foo");
true
> myObject["foo"] != null;
true
> myObject.foo != null;
true
> "foo" in myObject;
true
```

So if they all return the same thing, which one should we use? In order to answer that question, we need to talk about the JavaScript prototype chain.

## Prototype-based inheritance

JavaScript has a prototype based inheritance system, where rather than having a class definition that acts as a template, objects have a prototype that is another concrete object. (That is, there is no distinction between a class _definition_ and a class _instance_ in the prototype-based model.) It’s objects all the way down!

(Note that JavaScript as of ES5 has a notion of classes, but these are really syntactic sugar around the prototype model.)

The object literal syntax `const foo = { someProperty: “someValue” }` always inherits from a prototype named `Object.prototype`; similarly,

- array literals inherit from `Array.prototype`
- string literals inherit from `String.prototype`
- regular expression literals inherit from `RegExp.prototype`

```jsx
const someObject = { someProperty: "someValue" };
const someArray = [];
const someString = "hello, world";
const someRegexp = /.*/;

> Object.getPrototypeOf(someObject) === Object.prototype
true
> Object.getPrototypeOf(someArray) === Array.prototype
true
> Object.getPrototypeOf(someString) === String.prototype
true
> Object.getPrototypeOf(someRegexp) === RegExp.prototype
true
```

### Prototypes can be changed, even after object creation

The prototype of an object can be changed by:

- Setting the `__proto__` property of an object to something else
- Calling `Object.setPrototypeOf` on the object

This can be done **after** the object has already been created:

```jsx
const bar = { barProperty: 123 };
const foo = { fooProperty: 456 };

> foo.barProperty
undefined

foo.__proto__ = bar;
> foo.barProperty
123
```

Importantly, because prototypes are **themselves objects**, they can always be modified later. Yes, even `Object.prototype`!

```jsx
const foo = { fooProperty: 456 };

> foo.barProperty
undefined

Object.prototype.barProperty = 123;
> foo.barProperty
123
```

Hopefully this goes without saying: even if you’re _allowed_ do it, please don’t change `Object.prototype`! It will affect _every object literal_.

### The prototype chain

A prototype can also have a prototype of its own:

```jsx
const parent = { parentProperty: 'abc' };
const child = {
  childProperty: 'def',
  __proto__: parent
};

> Object.getPrototypeOf(child) === parent
true
> Object.getPrototypeOf(parent) === Object.prototype
true
> Object.getPrototypeOf(Object.prototype)
null
```

The term “prototype chain” comes from the act of following the prototype of an object, then the prototype of that prototype, and so on until `null` is reached, which does not have a prototype and acts as the end of the prototype chain. In this example, the prototype chain starting from `child` is:

`child` → `parent` → `Object.prototype` → `null` (end of prototype chain)

## Own properties vs. inherited properties

When we are asking if an object has a property, the JavaScript runtime might ask us back: “do you want to check inherited properties too?”

Properties that are inherited are, well, **inherited properties**, while properties that are not inherited are called **own properties**.

And when we try to access a property on an object using `myObject.someProperty`, we’re checking not just the object’s own properties, but all of its inherited properties too. This is called [climbing up the prototype chain](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain#inheritance_with_the_prototype_chain):

> JavaScript objects are dynamic "bags" of properties (referred to as **own properties**). JavaScript objects have a link to a prototype object. When trying to access a property of an object, the property will not only be sought on the object but on the prototype of the object, the prototype of the prototype, and so on until either a property with a matching name is found or the end of the prototype chain is reached.

## Retrying the test with inherited properties

What if our object literal inherited from another object? We’ll create a `customPrototype` object that inherits from `Object.prototype`, then make `myObject` inherit from `customPrototype`:

```jsx
const customPrototype = {
  baz: 500,
};

const myObject = {
  foo: "bar",
  meaningOfLife: 42,
  __proto__: customPrototype,
};

> Object.getPrototypeOf(myObject) === customPrototype;
true
```

And now let’s see what happens when we check each method against the property `baz`, which only exists on `customPrototype` and not in `myObject`:

```jsx
> myObject.hasOwnProperty('baz')
false
> Object.prototype.hasOwnProperty.call(myObject, 'baz')
false
> Object.hasOwn(myObject, 'baz')
false
> myObject['baz'] != null
true
> myObject.baz != null
true
> 'baz' in myObject
true
```

## Checking for own properties

So based on the above results, we can see that the following methods check if the property is present either on the object itself (an own property) or anywhere in its prototype chain (an inherited property):

- `myObject[someProperty]`
- `myObject.someProperty`
- `"someProperty" in myObject`

While these methods only check if the property is an own property:

- `myObject.hasOwnProperty("someProperty")`
- `Object.prototype.hasOwnProperty.call(myObject, "someProperty")`
- `Object.hasOwn(myObject, "someProperty")`

### The difference between these 3 own property checks

A natural question you’d ask is, what’s the difference between these three methods? Don’t they all check if the property is an own property?

Unfortunately, `hasOwnProperty` is not protected by JavaScript, which means that [an object can redefine it](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/hasOwnProperty#using_hasownproperty_as_a_property_name) to e.g. always return `false` (or if you’re truly malevolent, the opposite of what you would expect):

```jsx
const foo = {
  hasOwnProperty() {
    return false;
  },
  fooProperty: 123,
};

foo.hasOwnProperty("fooProperty");
> false

// or even more evil
const bar = {
  hasOwnProperty(key) {
    return !Object.prototype.hasOwnProperty.call(this, key);
  },
  barProperty: 123,
};

bar.hasOwnProperty("barProperty");
> false
bar.hasOwnProperty("shouldNotExist");
> true
```

To work around the possibility of an object redefining `hasOwnProperty`, we can use `Object.prototype.hasOwnProperty.call(myObject, someProperty)` instead, which is a bit safer [1]:

```jsx
const foo = {
  hasOwnProperty() {
    return false;
  },
  fooProperty: 123,
};

foo.hasOwnProperty("fooProperty");
> false
Object.prototype.hasOwnProperty.call(foo, "fooProperty")
> true

const bar = {
  hasOwnProperty(key) {
    return !Object.prototype.hasOwnProperty.call(this, key);
  },
  barProperty: 123,
};

bar.hasOwnProperty("barProperty");
> false
Object.prototype.hasOwnProperty.call(bar, "barProperty");
> true

bar.hasOwnProperty("shouldNotExist");
> true
Object.prototype.hasOwnProperty.call(bar, "shouldNotExist");
> false
```

Since this is rather verbose and inconvenient, `Object.hasOwn` was introduced [as a replacement for `Object.prototype.hasOwnProperty`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/hasOwn#description), because it cannot be redefined:

```jsx
const foo = {
  hasOwn() {
    // this doesn't affect Object.hasOwn at all
    return false;
  },
  fooProperty: 123,
};

Object.hasOwn(foo, "fooProperty");
> true
```

So `Object.hasOwn` allows us to avoid this potential pitfall [2]. The downside is that it was [added relatively recently](https://caniuse.com/mdn-javascript_builtins_object_hasown), so only more recent browsers support it: Chrome 93+, Edge 93+, Safari 15.4+, Firefox 92+, and no support for it in IE11. (You could of course transpile or [polyfill](https://github.com/zloirock/core-js#ecmascript-object) `Object.hasOwn` if needed.)

## Conclusion: what do I use?

If you’re only interested in **own properties** (in my experience, almost always what you _actually_ want):

- **Use `Object.hasOwn(myObject, someProperty)`**
- If you can’t use `Object.hasOwn` due to browser support requirements, use `Object.prototype.hasOwnProperty.call(myObject, someProperty)` instead

If you want to check **all** properties, including inherited properties, use any of the following:

- `myObject[someProperty]`
- `myObject.someProperty`
- `"someProperty" in myObject`

### Can I use indexing or `in` for own properties anyway?

If you are positive that

1. `myObject` is an object literal that inherits only from `Object.prototype`, and
2. no other code is modifying `Object.prototype`,

then indexing or `in` will also return the expected result.

It should be noted that even in these cases, if `someProperty` isn’t an own property of `myObject`, indexing or using `in` will still climb up the prototype chain and check properties unnecessarily, so using `Obect.hasOwn` is preferred.

---

[1] Note that I said _a bit_ safer, since even `Object.prototype.hasOwnProperty` can be redefined…

```jsx
Object.prototype.hasOwnProperty = () => false;
const foo = { bar: 123 };

> Object.prototype.hasOwnProperty(foo, "bar");
false
```

[2] Unfortunately, yes, `Object.hasOwn` can be redefined too:

```jsx
Object.hasOwn = () => false;
const foo = { bar: 123 };

> Object.hasOwn(foo, "bar");
false
```
