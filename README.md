# `%constructor%`.construct() method

## Problem

Quite a bit of existing javascript code uses old function based objects:

```js
function Foo(a, b, c) {
  this.a = a;
  this.b = b;
  this.c = c;
}

var foo = new Foo(1, 2, 3);
```

To enable inheritance, the prototype chain is manipulated, and such
pseudo-constructors are chained. For example, within Node.js:

```js
const util = require('util');

function Foo(a, b, c) {
  this.a = a;
  this.b = b;
  this.c = c;
}

function Bar(a, b, c) {
  Foo.call(a, b, c);
}
util.inherits(Bar, Foo);
```

With the introduction of Classes in the language, this pattern is broken.

While the following works:

```js
function A(a) {
  this.a = a;
}

class B extends A {
  constructor() {
    super(1);
  }
}

const b = new B();
console(b.a);     // 1
```

The following does not:

```js
class A {
  constructor(a) {
    this.a = a;
  }
}

function B() {
  A.call(1);
}
util.inherits(B, A);

const b = new B(); // Error!
```

This is because calling `A` without `new` or `super` is not permitted.

## Proposal: `{constructor}.construct(thisArg, arguments)` method

Constructor functions would expose a new `construct(thisArg, arguments)` method
that would allow the `[[Construct]]` function for the constructor function to
be called using the `thisArg` as the second argument, allowing the following
pattern:

```js
class A {
  constructor(a) {
    this.a = a;
  }
}

function B() {
  A.construct(this, 1);
}
util.inherits(B, A);

const b = new B();
console.log(b.a);   // 1
```

The `construct(thisArg[, ...args])` method would exist only on constructor
functions.

