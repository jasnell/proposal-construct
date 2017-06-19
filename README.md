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

const b = new B();  // Error
```

This is because calling `A` without `new` or `super` is not permitted.

The `Reflect.construct()` method has been suggested as a path forward, but
that only partially solves the problem, consider:

```js
const util = require('util');

class A {
  constructor(a) {
    this.a = a;
  }
}

///

function B(a) {
  if (!(this instanceof B))
    return new B(a);
  return Reflect.construct(A, [a], new.target);
}
util.inherits(B, A);

B.prototype.foo = function() {};

// This all works fine
const b = B(1);
console.log('b', b.a);
console.log('b', b.foo);
console.log('b', b instanceof B);
console.log('b', b instanceof A);

///

// This, however, does not

function C() {
  if (!(this instanceof C))
    return new C();
  B.call(this, 2);
}
util.inherits(C, B);

B.prototype.foo = function() {};


const c = C();
```

Here, the call to `C()` will throw because `new.target` in `B()` will be
`undefined`.

The example can be modified to account for the `undefined` `new.target`, but
doing so breaks the propagation of `this`:

```js
const util = require('util');

class A {
  constructor(a) {
    this.a = a;
  }
}

///

function B(a) {
  if (!(this instanceof B))
    return new B(a);
  return Reflect.construct(A, [a], new.target || this.constructor);
}
util.inherits(B, A);

B.prototype.foo = function() {};

// This all works fine
const b = B(1);
console.log('b', b.a);
console.log('b', b.foo);
console.log('b', b instanceof B);
console.log('b', b instanceof A);

///

// This, however, does not

function C() {
  if (!(this instanceof C))
    return new C();
  B.call(this, 2);
}
util.inherits(C, B);

B.prototype.foo = function() {};


const c = C();
console.log(c.a);   // undefined!
console.log(c.foo); // [Function]
```

Essentially, there is no way of propagating `this` from `C()` to the
constructor of `A`, even when using `Reflect.construct()` and `new.target`.

This poses a fundamental problem when attempting to update super classes
to use modern class syntax without breaking existing downstream subclass
code.

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
  if (!(this instanceof C))
    return new B();
  A.construct(this, 1);
}
util.inherits(B, A);

B.prototype.foo = function() {};

function C() {
  // Legacy style inheritance, very common in existing Node.js apps
  if (!(this instanceof C))
    return new C();
  B.call(this);
}
util.inherits(C, B);

const c = C();
console.log(c.a); // 1
c.foo();
```

The `construct(thisArg[, ...args])` method would exist only on constructor
functions.

## Alternative: Callable Constructors

Currently, class constructors are not callable. This alternative proposal would
make it possible to create a callable constructor decorator... which would work
essentially the same as old style inheritance with new class syntax, e.g.:

```js
class A {
  @callable constructor() {

  }
}

function B() {
  A.call(this);
}
util.inherits(B, A);

new B();
```

Here, the constructor `A()` is callable *only* if it is bound to `this`.

```js
A();           // Throws because there is no `this`
new A();       // Initializes `this`, calls the constructor
A.bind({})();  // Uses {} as this.
A.call({});    // Uses {} as this.
```

A class with a callable constructor is generally identical to other classes
in every other way with the notable exception that a class with a callable
constructor may only extend from a constructor function or another class with
a callable constructor.

That is, the following would result in a SyntaxError because `class A {}` does
not have a callable constructor:

```js
class A {}
class B {
  @callable constructor() {
    super();
  }
}
function C() {
  B.call(this);
}
new C();
```

The following, however, would work just fine:

```js
class A {
  @callable constructor() {}
}
class B {
  @callable constructor() {
    super();
  }
}
function C() {
  B.call(this);
}
new C();
```

The following would also work:

```js
class A {
  @callable constructor() {}
}

class B extends A {
  constructor() {
    super();
  }
}

new B();
```

