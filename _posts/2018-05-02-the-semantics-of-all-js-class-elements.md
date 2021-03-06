---
layout: post
title: The Semantics of All JS Class Elements
---

{{ page.title }}
================

This article summarizes the current and proposed class fields and methods semantics. With the exception of private static fields and methods at the time of writing, all the described semantics enjoy current <span class="nnum">TC39</span> consensus.

I will talk about the following table of features. The full feature set is the product of the columns.

| Visibility | Placement | Class Element |
|------------|-----------|---------------|
| Public     | Instance  | Field         |
| Private    | Static    | Method        |

<table class="narrow">
<tbody>
<tr>
<td rowspan="2" style="font-weight: bold">Visibility</td>
<td>Public</td>
</tr>
<tr>
<td>Private</td>
</tr>
<tr>
<td rowspan="2" style="font-weight: bold">Placement</td>
<td>Instance</td>
</tr>
<tr>
<td>Static</td>
</tr>
<tr>
<td rowspan="2" style="font-weight: bold">Class Element</td>
<td>Field</td>
</tr>
<tr>
<td>Method</td>
</tr>
</tbody>
</table>

I will describe the semantics of each column in detail, as well as the consequences of their combinations with each other and with existing JS language features.

Public instance and static methods are already shipping with ES6. The rest of the above feature matrix is being advanced by 3 individual proposals:

- [Public and private instance fields](https://github.com/tc39/proposal-class-fields/) at <span class="onum">stage 3</span>
- [Private instance methods](https://github.com/tc39/proposal-private-methods/) at <span class="onum">stage 3</span>
- [Public and private static fields, private static methods](https://github.com/tc39/proposal-static-class-features/) at <span class="onum">stage 2</span>

## Mental Model

Here is the mental model for the columns above. These intuitions will be bolstered by a deep dive into the semantics with concrete examples in the next section.

### Visibility

A **public** element is a writable and configurable property. As a property, public things participate in prototype inheritance.

A **private** element is that which is accessible via lexically scoped names only on objects of distinguished provenance. Since they are not properties, they also do not participate in prototype inheritance.

Thought of another way, the behavior of private is a strict subset of the behavior of public. Encapsulation is enabled precisely by combination of lexical scoping with the provenance restriction on object dispatch. The JS concept of private is designed for JS and differs from "private" in other languages.

### Placement

An **instance** element is accessed on instances of a class.

A **static** element is accessed on class constructors.

### Class Element

A **field** is state associated with a class that has a blessed declarative syntax. An instance field is per-instance state. A static field is per-class constructor state.

A **method** is some behavior associated with a class that has a blessed declarative syntax. Both instance and static methods have one identity per-class evaluation. Instance methods are on the prototype, and static methods are on the class constructor.

## Let's Make the Semantics Concrete with Examples

I hope to make clear the semantics of all class elements below with examples. Each example will be followed by a short explanation of the highlighted semantics.

### Public instance fields

Classes may declare public fields accessible as properties on instances.

<a class="permanchor"></a>
Public instance fields are properties added by `Object.defineProperty`. They are added at construction time of the object for the base class, before the constructor body runs.

<div class="wide">
{% highlight js %}
class Ex1 {
  publicField;

  constructor() {
    let desc = Object.getOwnPropertyDescriptor(this, "publicField");
    assert(desc.value === undefined);
    assert(desc.writable);
    assert(desc.enumerable);
    assert(desc.configurable);
  }
}

new Ex1;
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex1 {
 publicField;

 constructor() {
  let desc =
   Object.getOwnPropertyDescriptor(
    this,
    "publicField");
   assert(desc.value === undefined);
   assert(desc.writable);
   assert(desc.enumerable);
   assert(desc.configurable);
 }
}

new Ex1;
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
For subclasses, `this` throws `ReferenceError` if touched until `super()` is called.<sup><a href="#tdz" id="tdz-use">(*)</a></sup> So, public instance fields are added when `super()` returns.

<div class="wide">
{% highlight js %}
class Ex2_Base {
  basePublicField;
}

class Ex2_Sub extends Ex2_Base {
  subPublicField;

  constructor() {
    super();

    assert(this.hasOwnProperty("basePublicField"));
    assert(this.hasOwnProperty("subPublicField"));
  }
}

new Ex2_Sub;
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex2_Base {
 basePublicField;
}

class Ex2_Sub extends Ex2_Base {
 subPublicField;

 constructor() {
  super();

  assert(this.hasOwnProperty(
   "basePublicField"
  ));
  assert(this.hasOwnProperty(
   "subPublicField"
  ));
 }
}

new Ex2_Sub;
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
All constructors in JavaScript can return a different object, overriding the result from `new` and the newly bound `this` value from `super()`. For instance fields, if the super constructor returns something different, fields from the subclass are still added.

<div class="wide">
{% highlight js %}
class Ex3_Base {
  basePublicField;
}

class Ex3_ReturnTrickBase {
  trickyBasePublicField;

  constructor() {
    return new Ex3_Base;
  }
}

class Ex3_ReturnTrickSub extends Ex3_ReturnTrickBase {
  trickySubPublicField;

  constructor() {
    super();

    assert(!this.hasOwnProperty("trickyBasePublicField"));
    assert(this.hasOwnProperty("basePublicField"));
    assert(this.hasOwnProperty("trickySubPublicField"));
  }
}

new Ex3_ReturnTrickSub;
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex3_Base {
 basePublicField;
}

class Ex3_ReturnTrickBase {
 trickyBasePublicField;

 constructor() {
  return new Ex3_Base;
 }
}

class Ex3_ReturnTrickSub
extends Ex3_ReturnTrickBase {
 trickySubPublicField;

 constructor() {
  super();

  assert(!this.hasOwnProperty(
   "trickyBasePublicField"));
  assert(this.hasOwnProperty(
   "basePublicField"));
  assert(this.hasOwnProperty(
   "trickySubPublicField"));
 }
}

new Ex3_ReturnTrickSub;
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
Public field names may be computed, like properties. They are evaluated once per class evaluation.

<div class="wide">
{% highlight js %}
let count = 0;
function makeEx4(sym) {
  return class Ex4 {
    [(count++, sym)];
  };
}

let key = Symbol("key");
let Ex4 = makeEx4(key);
assert(count === 0);
let ex4a = new Ex4;
assert(ex4a.hasOwnProperty(key));
assert(count === 1);
let ex4b = new Ex4;
assert(ex4b.hasOwnProperty(key));
assert(count === 1);
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
let count = 0;
function makeEx4(sym) {
 return class Ex4 {
  [(count++, sym)];
 };
}

let key = Symbol("key");
let Ex4 = makeEx4(key);
assert(count === 0);
let ex4a = new Ex4;
assert(ex4a.hasOwnProperty(key));
assert(count === 1);
let ex4b = new Ex4;
assert(ex4b.hasOwnProperty(key));
assert(count === 1);
{% endhighlight %}
</div>

### Public instance methods

Classes may declare public methods accessible on instances via the prototype.

<a class="permanchor"></a>
Public instance methods are added to the class prototype with `Object.defineProperty` at class evaluation time. They are writable, non-enumerable, and configurable.

<div class="wide">
{% highlight js %}
class Ex5 {
  publicMethod() { return 42; }
}

let desc =  Object.getOwnPropertyDescriptor(Ex5.prototype, "publicMethod");
assert(desc.value === Ex5.prototype.publicMethod);
assert(desc.writable);
assert(!desc.enumerable);
assert(desc.configurable);
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex5 {
 publicMethod() { return 42; }
}

let proto = Ex5.prototype;
let desc =
 Object.getOwnPropertyDescriptor(
  proto,
  "publicMethod");
assert(
  desc.value === proto.publicMethod);
assert(desc.writable);
assert(!desc.enumerable);
assert(desc.configurable);
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
Generator function, async function, and async generator function forms may also be public instance methods.

<div class="wide">
{% highlight js %}
class Ex6 {
  *publicGeneratorMethod() { }
  async publicAsyncMethod() { }
  async *publicAsyncGeneratorMethod() { }
}
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex6 {
 *publicGeneratorMethod() { }
 async publicAsyncMethod() { }
 async *publicAsyncGeneratorMethod()
 { }
}
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
Getters and setters are possible as well. There are no generator, async, or async generator getter and setter forms.

<div class="wide">
{% highlight js %}
class Ex7 {
  get publicAccessor() { return 42; }
  set publicAccessor(x) { }
}

let desc = Object.getOwnPropertyDescriptor(Ex7.prototype, "publicAccessor");
assert(desc.value === undefined);
assert(!desc.enumerable);
assert(desc.configurable);
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex7 {
 get publicAccessor() { return 42; }
 set publicAccessor(x) { }
}

let desc =
 Object.getOwnPropertyDescriptor(
  Ex7.prototype,
  "publicAccessor");
assert(desc.value === undefined);
assert(!desc.enumerable);
assert(desc.configurable);
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
In instance methods, `super` references the superclass's `prototype` property. The following invokes `Base.prototype.basePublicMethod`.<sup><a href="#homeobj" id="homeobj-use">(&dagger;)</a></sup>

<div class="wide">
{% highlight js %}
class Ex8_Base {
  basePublicMethod() { return 42; }
}

class Ex8_Sub extends Ex8_Base {
  subPublicMethod() {
    return super.basePublicMethod();
  }
}

assert((new Ex8_Sub).subPublicMethod() === 42);
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex8_Base {
 basePublicMethod() { return 42; }
}

class Ex8_Sub extends Ex8_Base {
 subPublicMethod() {
  return super.basePublicMethod();
 }
}

let ex8 = new Ex8_Sub;
assert(ex8.subPublicMethod() === 42);
{% endhighlight %}
</div>

### Private instance fields

Classes may declare private fields accessible on base class or subclass instances from inside the lexical scope of the class declaration itself.

<a class="permanchor"></a>
Private instance fields are declared with **`#` names** (said "hash names"), identifiers that are prefixed with `#`. Though a different character, this follows the convention of signaling privacy with `_`-prefixed property names.

<center><span style="font-variant: small-caps; font-size: 150%; margin-right: 0.4em;"><span style="font: bold 75% 'Cousine', monospace">#</span> is the new <span style="font: bold 75% 'Cousine', monospace">_</span></span>,</center>

with encapsulation being enforced by the language instead of by convention. <code>#</code> is part of the name itself and is used in both in declaration and access.

<div class="wide">
{% highlight js %}
class Ex9 {
  #privateField;

  constructor() {
    this.#privateField = 42;
  }
}

new Ex9;
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex9 {
 #privateField;

 constructor() {
  this.#privateField = 42;
 }
}

new Ex9;
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
The lexical scoping rules of `#` names are stricter than those of identifier names. It is a syntax error to refer to `#` names that are not in scope.

<div class="wide">
{% highlight js %}
class Ex10_A {
  #privateField;

  constructor() {
    this.#nonExistentField = 42; // Syntax error
  }
}
{% endhighlight %}
{% highlight js %}
class Ex10_B {
  #privateField;
}

(new Ex10_B).#privateField; // Syntax error
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex10_A {
 #privateField;

 constructor() {
  // Syntax error
  this.#nonExistentField = 42;
 }
}
{% endhighlight %}
{% highlight js %}
class Ex10_B {
 #privateField;
}

// Syntax error
(new Ex10_B).#privateField;
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
Like lexical bindings, it is a syntax error to have multiple same-named `#` names in the same scope (i.e. the class declaration), while shadowing via nested scopes is allowed.

Note that since all `#` names start with `#` and property names cannot start with `#`, the two cannot be in conflict.

<div class="wide">
{% highlight js %}
class Ex11_A {
  #privateField;
  #privateField; // Syntax error
}
{% endhighlight %}
{% highlight js %}
class Ex11_Outer {
  #privateField;

  constructor {
    class Ex11_Inner {
      #privateField;

      privateFieldValue() {
        return this.#privateField;
      }

      constructor() {
        this.#privateField = 42;
      }
    }

    assert((new Ex11_Inner).privateFieldValue() === 42);
    assert(this.#privateField === undefined);
  }
}

new Ex11_Outer;
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex11_A {
 #privateField;
 #privateField; // Syntax error
}
{% endhighlight %}
{% highlight js %}
class Ex11_Outer {
 #privateField;

 constructor {
  class Ex11_Inner {
   #privateField;

   privateFieldValue() {
    return this.#privateField;
   }

   constructor() {
    this.#privateField = 42;
   }
  }

  let ex11 = new Ex11_Inner;
  assert(
   ex11.privateFieldValue() === 42);
  assert(
   this.#privateField === undefined);
 }
}

new Ex11_Outer;
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
It is also a syntax error to `delete` `#` names.

<div class="wide">
{% highlight js %}
class Ex12 {
  #privateField;
  constructor() {
    delete this.#privateField; // Syntax error
  }
}
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex12 {
 #privateField;
 constructor() {
  // Syntax error
  delete this.#privateField;
 }
}
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
Private field accesses are strictly more restrictive than public field accesses. Getting a `#` name on an object that doesn't have a private field with that name is a `TypeError`. The same goes for setting a `#` name. Private fields are not properties, and do not participate in the machinery of property lookup like prototype inheritance. This is to enable encapsulation, as property lookup machinery is observable, e.g., by `Proxy`.

Private fields combines lexical scoping with a provenance restriction on dispatch. For private instance fields, the provenance is having been constructed, either as a base class or a subclass, by the class that declared the private instance field.

<div class="wide">
{% highlight js %}
class Ex13 {
  #privateField;

  getField() { return this.#privateField; }
  setField(v) { this.#privateField = v; }
}

let ex13 = new Ex13;
let onProto = Object.create(ex13);
assertThrows(() => ex13.getField.call({}), TypeError);
assertThrows(() => ex13.setField.call({}, 42), TypeError);
assertThrows(() => ex13.getField.call(onProto), TypeError);
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex13 {
 #privateField;

 getField() {
  return this.#privateField;
 }
 setField(v) {
  this.#privateField = v;
 }
}

let ex13 = new Ex13;
let onProto = Object.create(ex13);
assertThrows(
 () => ex13.getField.call({}),
 TypeError);
assertThrows(
 () => ex13.setField.call({}, 42),
 TypeError);
assertThrows(
 () => ex13.getField.call(onProto),
 TypeError);
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
`#` names are accessible through direct `eval`, like other lexically scoped things.

<div class="wide">
{% highlight js %}
class Ex14 {
  #privateField;

  constructor() {
    eval("this.#privateField = 42");
  }
}

new Ex14;
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex14 {
 #privateField;

 constructor() {
  eval("this.#privateField = 42");
 }
}

new Ex14;
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
Private fields are added at the same time as public fields, either at construction time in the base class, or after `super()` returns in a subclass.

<div class="wide">
{% highlight js %}
class Ex15_Base {
  #basePrivateField;
}

class Ex15_Sub extends Ex15_Base {
  #subPrivateField;

  constructor() {
    super();

    this.#basePrivateField = 42;
    this.#subPrivateField = 84;
  }
}

new Ex15_Sub;
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex15_Base {
 #basePrivateField;
}

class Ex15_Sub extends Ex15_Base {
 #subPrivateField;

 constructor() {
  super();

  this.#basePrivateField = 42;
  this.#subPrivateField = 84;
 }
}

new Ex15_Sub;
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
Like with public instance fields, if the `super()` overrides the return value, private fields from the subclass are still added. For implementers, this means private fields may be added to arbitrary objects.

<div class="wide">
{% highlight js %}
class Ex16_ReturnTrickBase {
  constructor() {
    return new Proxy({}, {});
  }
}

class Ex16_ReturnTrickSub extends Ex16_ReturnTrickBase {
  #subPrivateField;

  constructor() {
    super();

    this.#subPrivateField = 42;
  }
}

new Ex16_ReturnTrickSub;
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex16_ReturnTrickBase {
 constructor() {
  return new Proxy({}, {});
 }
}

class Ex16_ReturnTrickSub
extends Ex16_ReturnTrickBase {
 #subPrivateField;

 constructor() {
  super();

  this.#subPrivateField = 42;
 }
}

new Ex16_ReturnTrickSub;
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
While `#` names are not first class in the language, they have observably distinct per-class evaluation identities. One evaluation of a class declaration cannot access the private fields of another evaluation of the same declaration.

<div class="wide">
{% highlight js %}
function makeEx17() {
  return class Ex17 {
    #privateField;

    getField() { return this.#privateField; }
  };
}

let ex17a = new makeEx17();
let ex17b = new makeEx17();
assertThrows(() => ex17a.getField.call(ex17b), TypeError);
assertThrows(() => ex17b.getField.call(ex17a), TypeError);
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
function makeEx17() {
 return class Ex17 {
  #privateField;

  getField() {
   return this.#privateField;
  }
 };
}

let ex17a = new makeEx17();
let ex17b = new makeEx17();
assertThrows(
 () => ex17a.getField.call(ex17b),
 TypeError);
assertThrows(
 () => ex17b.getField.call(ex17a),
 TypeError);
{% endhighlight %}
</div>

Finally, note that there is currently no shorthand notation for accessing `#` names. Their access requires a receiver.

### Private instance methods

Private instance methods are analogous to public instance methods. Their access is restricted in the same fashion as private instance fields.

<a class="permanchor"></a>
These methods are specified as non-writable private fields of class instances. Like private instance fields, they are added at construction time for the base classes and after `super()` returns for subclasses.

<div class="wide">
{% highlight js %}
class Ex18 {
  #privateMethod() { return 42; }

  constructor() {
    assert(this.#privateMethod() === 42);
    assertThrows(() => this.#privateMethod = null, TypeError);
  }
}

new Ex18;
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex18 {
 #privateMethod() { return 42; }

 constructor() {
  assert(
   this.#privateMethod() === 42);
  assertThrows(
   () => this.#privateMethod = null,
   TypeError);
 }
}

new Ex18;
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
Generator function, async function, and async generator function forms may also be private instance methods.

<div class="wide">
{% highlight js %}
class Ex19 {
  *#privateGeneratorMethod() { }
  async #privateAsyncMethod() { }
  async *#privateAsyncGeneratorMethod() { }
}
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex19 {
 *#privateGeneratorMethod() { }
 async #privateAsyncMethod() { }
 async *#privateAsyncGeneratorMethod()
 { }
}
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
Getters and setters are possible as well. There are no generator, async, or async generator getter and setter forms for private instance methods.

Private methods are specified as a per-instance list of map of `#` names to property descriptors. This preserves the symmetry of expressible forms with public instance methods, as well as enforce the restrictions that come with private fields.

<div class="wide">
{% highlight js %}
class Ex20 {
  get #privateAccessor() { return 42; }
  set #privateAccessor(x) { }

  constructor() {
    assert(this.#privateAccessor === 42);
    this.#privateAccessor = "ignored";
  }
}

new Ex20;
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex20 {
 get #privateAccessor() {
  return 42;
 }
 set #privateAccessor(x) { }

 constructor() {
  assert(
  this.#privateAccessor === 42);
  this.#privateAccessor = "ignored";
 }
}

new Ex20;
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
There is a single function identity per class-evaluation for private instance methods. Even though they are specified as per-instance private fields, instance methods are shared across all instances.

<div class="wide">
{% highlight js %}
let exfiltrated;
class Ex21 {
  #privateMethod() { }

  constructor() {
    if (exfiltrated === undefined) {
      exfiltrated = this.#privateMethod;
    }
    assert(exfiltrated === this.#privateMethod);
  }
}

new Ex21;
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
let exfil;
class Ex21 {
 #privateMethod() { }

 constructor() {
  if (exfil === undefined) {
   exfil = this.#privateMethod;
  }
  assert(
   exfil === this.#privateMethod);
 }
}

new Ex21;
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
In private instance methods, `super` and `this` follow the same semantics as those in public instance methods. Since private fields and methods are not added to the prototype, `#privateMethod()` throws below. Similarly, when an object of incompatible provenance is passed as the receiver to an instance private method, the private field lookup throws.

<div class="wide">
{% highlight js %}
class Ex22_Base {
  #privateMethod() { return 42; }
}

class Ex22_Sub extends Ex22_Base {
  #privateMethod() {
    assertThrows(() => super.#privateMethod(), TypeError);
  }

  #privateMethodTwo() {
    this.#privateMethod();
  }

  publicMethod() {
    this.#privateMethodTwo();
  }

  constructor() {
    this.#privateMethod();
  }
}

assertThrows(() => (new Ex22_Sub).publicMethod.call({}), TypeError);
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex22_Base {
 #privateMethod() { return 42; }
}

class Ex22_Sub extends Ex22_Base {
 #privateMethod() {
  assertThrows(
   () => super.#privateMethod(),
   TypeError);
 }

 #privateMethodTwo() {
  this.#privateMethod();
 }

 publicMethod() {
  this.#privateMethodTwo();
 }

 constructor() {
  this.#privateMethod();
 }
}

let ex22 = new Ex22_Sub;
assertThrows(
 () => ex22.publicMethod.call({}),
 TypeError);
{% endhighlight %}
</div>

### Instance field initializers

All fields can be initialized with an initializer expression in situ with the declaration. Initializers are run in declaration order. Instance field initializer expressions are specified as the bodies of non-observable instance methods, which informs the values of `this`, `new.target`, and `super`.

<a class="permanchor"></a>
Fields without initializers are initialized to `undefined`.

<div class="wide">
{% highlight js %}
class Ex23 {
  #privateField = 42;
  publicFieldOne = 84;
  publicFieldTwo;

  constructor() {
    assert(this.#privateField === 42);
    assert(this.publicFieldOne === 84);
    assert(this.publicFieldTwo === undefined);
  }
}

new Ex23;
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex23 {
 #privateField = 42;
 publicFieldOne = 84;
 publicFieldTwo;

 constructor() {
  assert(
   this.#privateField === 42);
  assert(
   this.publicFieldOne === 84);
  assert(
   this.publicFieldTwo === undefined);
 }
}

new Ex23;
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
Field initializers are run as fields are added, in declaration order, and this order is observable by the initializers. `#privateField` is `false` in the following as `publicField` has has not been added yet when evaluating `#privateField`'s initializer. There is also no error when initializing `publicField` with `this.#privateField`, owing to its coming after `#privateField`.

These initializers are specified as methods that return the result of the initializer expressions. The methods need not be reified in implementation, but this fiction of specification informs the values of `this`. In instance field initializers, `this` is the object under construction. By the time the instance field initializer runs, `this` is accessible.

<div class="wide">
{% highlight js %}
class Ex24 {
  #privateField = this.hasOwnProperty("publicField");
  publicField = this.#privateField;

  constructor() {
    assert(!this.#privateField);
    assert(!this.publicField);
  }
}

new Ex24;
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex24 {
 #privateField =
  this.hasOwnProperty("publicField");
 publicField = this.#privateField;

 constructor() {
  assert(!this.#privateField);
  assert(!this.publicField);
 }
}

new Ex24;
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
Multiple same-named public fields and methods are allowed in the same class declaration, and their initializers are run in order. Since public methods and fields are properties added with `Object.defineProperty`, the last field or method overrides all previous ones.

This does not happen with private fields and methods, as multiple same-named `#` names in the same scope is a syntax error.

<div class="wide">
{% highlight js %}
let log = ""
class Ex25 {
  publicField = (log += "1", 42);
  publicField = (log += "2", 84);
  publicField() { }
}

assert(typeof (new Ex25).publicField === "function");
assert(log === "12");
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
let log = ""
class Ex25 {
  publicField = (log += "1", 42);
  publicField = (log += "2", 84);
  publicField() { }
}

let ex25 = new Ex25;
assert(
 typeof ex25.publicField ===
 "function");
assert(log === "12");
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
Instance methods are added before any initializer is run, so all instance methods are available in instance field initializers.

<div class="wide">
{% highlight js %}
class Ex26 {
  #privateField = this.#privateMethod();
  publicField = this.#privateField;

  #privateMethod() { return 42; }
}

assert((new Ex26).publicField === 42);
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex26 {
 #privateField =
  this.#privateMethod();
 publicField = this.#privateField;

 #privateMethod() { return 42; }
}

let ex26 = new Ex26;
assert(ex26.publicField === 42);
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
`new.target` is `undefined` in field initializers.

<div class="wide">
{% highlight js %}
class Ex27 {
  publicField = new.target;
}

assert((new Ex27).publicField === undefined);
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex27 {
 publicField = new.target;
}

let ex27 = new Ex27;
assert(
 ex27.publicField === undefined);
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
As in instance methods, `super` references the superclass's prototype in instance field initializers.

<div class="wide">
{% highlight js %}
class Ex28_Base {
  baseMethod() { return 42; }
}

class Ex28_Sub extends Ex28_Base {
  subPublicField = super.baseMethod();
}

assert((new Ex28_Sub).subPublicField === 42);
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex28_Base {
 baseMethod() { return 42; }
}

class Ex28_Sub extends Ex28_Base {
 subPublicField = super.baseMethod();
}

let ex28 = new Ex28_Sub;
assert(ex28.subPublicField === 42);
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
Class bodies are always strict code, so initializers cannot leak any new bindings via direct `eval`.

<div class="wide">
{% highlight js %}
class Ex29 {
  publicField = eval("var x = 42;");
}

new Ex29;
assertThrows(() => x, ReferenceError);
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex29 {
 publicField = eval("var x = 42;");
}

new Ex29;
assertThrows(
 () => x,
 ReferenceError);
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
Use of `arguments` is a syntax error in field initializers.

<div class="wide">
{% highlight js %}
// Syntax error
class Ex30 {
  publicField = arguments;
}
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
// Syntax error
class Ex30 {
 publicField = arguments;
}
{% endhighlight %}
</div>

### Public static fields

Classes may declare public static fields, which are accessible as properties on the class constructor.

<a class="permanchor"></a>
Public static fields are added to the class constructor with `Object.defineProperty` at class evaluation time. Their semantics are otherwise identical to public instance fields.

<div class="wide">
{% highlight js %}
class Ex31 {
  static PUBLIC_STATIC_FIELD;
}

let desc = Object.getOwnPropertyDescriptor(Ex31, "PUBLIC_STATIC_FIELD");
assert(desc.value === undefined);
assert(desc.writable);
assert(desc.enumerable);
assert(desc.configurable);
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex31 {
 static PUBLIC_STATIC_FIELD;
}

let desc =
 Object.getOwnPropertyDescriptor(
  Ex31,
  "PUBLIC_STATIC_FIELD"
 );
assert(desc.value === undefined);
assert(desc.writable);
assert(desc.enumerable);
assert(desc.configurable);
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
Public static fields are only initialized on the class in which they are defined, not reinitialized on subclasses. Subclass constructors have their superclasses as their prototype. Public static fields of superclasses are accessed via the prototype chain.

<div class="wide">
{% highlight js %}
class Ex32_Base {
  static BASE_PUBLIC_STATIC_FIELD;
}

class Ex32_Sub extends Ex32_Base {
  static SUB_PUBLIC_STATIC_FIELD;
}

assert(Ex32_Base.hasOwnProperty("BASE_PUBLIC_STATIC_FIELD"));
assert(Ex32_Sub.hasOwnProperty("SUB_PUBLIC_STATIC_FIELD"));
assert(Object.getPrototypeOf(Ex32_Sub) === Ex32_Base);
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex32_Base {
 static BASE_PUBLIC_STATIC_FIELD;
}

class Ex32_Sub extends Ex32_Base {
 static SUB_PUBLIC_STATIC_FIELD;
}

assert(
 Ex32_Base.hasOwnProperty(
  "BASE_PUBLIC_STATIC_FIELD"));
assert(
 Ex32_Sub.hasOwnProperty(
  "SUB_PUBLIC_STATIC_FIELD"));
assert(
 Object.getPrototypeOf(Ex32_Sub) ===
 Ex32_Base);
{% endhighlight %}
</div>

### Public static methods

Public static methods are declared function forms and, like public static fields, are also accessible as properties on the class constructor. Also like public instance methods, generator function, async function, async generator function, getter, and setter forms are accepted.

<a class="permanchor"></a>
These methods are added to the class constructor with `Object.defineProperty` at class evaluation time. Like public instance methods, they are writable, non-enumerable, and configurable.

<div class="wide">
{% highlight js %}
class Ex33 {
  static publicStaticMethod() { return 42; }
}

let desc = Object.getOwnPropertyDescriptor(Ex33, "publicStaticMethod");
assert(desc.value === Ex33.publicMethod);
assert(desc.writable);
assert(!desc.enumerable);
assert(desc.configurable);
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex33 {
 static publicStaticMethod() {
  return 42;
 }
}

let desc =
 Object.getOwnPropertyDescriptor(
  Ex33,
  "publicStaticMethod");
assert(
 desc.value === Ex33.publicMethod);
assert(desc.writable);
assert(!desc.enumerable);
assert(desc.configurable);
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
In static methods, `super` references the superclass constructor, and the superclass's public static methods are accessible.<sup><a href="#homeobj2" id="homeobj-use2">(&Dagger;)</a></sup>

<div class="wide">
{% highlight js %}
class Ex34_Base {
  static basePublicStaticMethod() { return 42; }
}

class Ex34_Sub extends Ex34_Base {
  static subPublicStaticMethod() {
    return super.basePublicStaticMethod();
  }
}

assert(Ex34_Sub.subPublicStaticMethod() === 42);
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex34_Base {
 static basePublicStaticMethod() {
  return 42;
 }
}

class Ex34_Sub extends Ex34_Base {
 static subPublicStaticMethod() {
  return super.
         basePublicStaticMethod();
 }
}

assert(
 Ex34_Sub.subPublicStaticMethod() ===
 42);
{% endhighlight %}
</div>

### Private static fields

Classes may declare private fields accessible on the class constructor from inside the lexical scope of the class declaration itself.

<a class="permanchor"></a>
Private static fields are added to the class constructor at class evaluation time.

<div class="wide">
{% highlight js %}
class Ex35 {
  static #PRIVATE_STATIC_FIELD;

  static publicStaticMethod() {
    Ex35.#PRIVATE_STATIC_FIELD = 42;
    return Ex35.#PRIVATE_STATIC_FIELD;
  }
}

assert(Ex35.publicStaticMethod() === 42);
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex35 {
 static #PRIVATE_STATIC_FIELD;

 static publicStaticMethod() {
  Ex35.#PRIVATE_STATIC_FIELD = 42;
  return Ex35.#PRIVATE_STATIC_FIELD;
 }
}

assert(
 Ex35.publicStaticMethod() === 42);
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
The provenance restriction of private static fields restricts access to the class constructor. The following throws because the `this` value of `basePublicStaticMethod` is the subclass constructor, which does not have the `#BASE_PRIVATE_STATIC_FIELD` field. This is the natural result, though unexpected for some, of the composition of the private field provenance restriction with `this` dynamicity.

This type of error can be avoided by always using the class constructor as the receiver when accessing private static elements.

<div class="wide">
{% highlight js %}
class Ex36_Base {
  static #BASE_PRIVATE_STATIC_FIELD;

  static basePublicStaticMethod() {
    return this.#BASE_PRIVATE_STATIC_FIELD;
  }
}

class Ex36_Sub extends Ex36_Base { }

assertThrows(() => Ex36_Sub.basePublicStaticMethod(), TypeError);
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex36_Base {
 static #BASE_PRIVATE_STATIC_FIELD;

 static basePublicStaticMethod() {
  return this.
         #BASE_PRIVATE_STATIC_FIELD;
 }
}

class Ex36_Sub extends Ex36_Base { }

assertThrows(
 () => Ex36_Sub.
       basePublicStaticMethod(),
 TypeError);
{% endhighlight %}
</div>

### Private static methods

Private static methods are analogous to public static methods. Their access is restricted in the same fashion as private static fields.

<a class="permanchor"></a>
These methods are specified as non-writable private fields of class constructors. Like private static fields, they are added at class evaluation time.

Like all methods, generator function, async function, async generator function, getter, and setter forms are accepted.

<div class="wide">
{% highlight js %}
class Ex37 {
  static #privateStaticMethod() { return 42; }

  constructor() {
    assertThrows(() => Ex37.#privateStaticMethod = null, TypeError);
    assert(Ex37.#privateStaticMethod() === 42);
  }
}

new Ex37;
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex37 {
 static #privateStaticMethod() {
  return 42;
 }

 constructor() {
  assertThrows(
   () => Ex37.#privateStaticMethod =
         null,
   TypeError);
  assert(
   Ex37.#privateStaticMethod() ===
   42);
 }
}

new Ex37;
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
Like public static methods, `super` references the superclass constructor.

<div class="wide">
{% highlight js %}
class Ex38_Base {
  static basePublicStaticMethod() { return 42; }
}

class Ex38_Sub extends Ex38_Base {
  static #subStaticPrivateMethod() {
    assert(super.basePublicStaticMethod() === 42);
  }

  static check() {
    Ex38_Sub.#subStaticPrivateMethod();
  }
}

Ex38_Sub.check();
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex38_Base {
 static basePublicStaticMethod() {
  return 42;
 }
}

class Ex38_Sub extends Ex38_Base {
 static #subStaticPrivateMethod() {
  assert(
   super.basePublicStaticMethod() ===
   42);
 }

 static check() {
   Ex38_Sub.#subStaticPrivateMethod();
 }
}

Ex38_Sub.check();
{% endhighlight %}
</div>

### Static field initializers

Static fields may have in-situ initializers. Like instance field initializers, they are also run in declaration order, and the expressions are specified as bodies of non-observable static methods.

Like instance field initializers, static methods are added before any initializer is run, so all static methods are available in static field initializers.

<a class="permanchor"></a>
As initializer expressions are specified as static method bodies, `super` references the superclass constructor, and `this` references the class constructor.

<div class="wide">
{% highlight js %}
class Ex39_Base {
  static BASE_PUBLIC_STATIC_FIELD = this;
  static basePublicStaticMethod() { return 42; }
}

class Ex39_Sub extends Ex39_Base {
  static SUB_PUBLIC_STATIC_FIELD = super.basePublicStaticMethod();
}

assert(Ex39_Sub.BASE_PUBLIC_STATIC_FIELD === Ex39_Base);
assert(Ex39_Sub.SUB_PUBLIC_STATIC_FIELD === 42);
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex39_Base {
 static BASE_PUBLIC_STATIC_FIELD =
  this;
 static basePublicStaticMethod() {
  return 42;
 }
}

class Ex39_Sub extends Ex39_Base {
 static SUB_PUBLIC_STATIC_FIELD =
  super.basePublicStaticMethod();
}

assert(
 Ex39_Sub.BASE_PUBLIC_STATIC_FIELD ===
 Ex39_Base);
assert(
 Ex39_Sub.SUB_PUBLIC_STATIC_FIELD ===
 42);
{% endhighlight %}
</div>

* * *

<a class="permanchor"></a>
By the time static field initializers are evaluated, the class name binding inside the class scope is initialized (i.e. may be accessed and does not throw a `ReferenceError`).

<div class="wide">
{% highlight js %}
class Ex40 {
  static #PRIVATE_STATIC_FIELD = 42;
  static PUBLIC_STATIC_FIELD = Ex40.#PRIVATE_STATIC_FIELD;
}

assert(Ex40.PUBLIC_STATIC_FIELD === 42);
{% endhighlight %}
</div>

<div class="narrow">
{% highlight js %}
class Ex40 {
 static #PRIVATE_STATIC_FIELD = 42;
 static PUBLIC_STATIC_FIELD =
  Ex40.#PRIVATE_STATIC_FIELD;
}

assert(
 Ex40.PUBLIC_STATIC_FIELD === 42);
{% endhighlight %}
</div>

## Some Take Aways

We've explored the semantics of all JavaScript class elements. Perhaps some semantics were surprising, while others were expected. I believe a common source of mismatched intuition for private fields is the provenance restriction on `#` names, and I hope this article was helpful in making this clearer. In closing, I offer these two aphorisms.<img id="seal" src="{{ site.baseurl }}/seal.svg" alt="seal">

<div class="takeaways">
<style scoped>
.takeaways {
    text-align: center;
}
.takeaway {
    font-variant: small-caps;
    font-size: 150%;
}

@media (max-width: 1066px) {
    .takeaways {
        margin: 1em 0;
    }
    .takeaway {
        line-height: 14px;
    }
}
</style>
<span class="takeaway"><span style="font: bold 75% 'Cousine', monospace">#</span> is the new <span style="font: bold 75% 'Cousine', monospace">_</span></span>
<br>
and
<br>
<span class="takeaway">private fields have a provenance restriction</span>
</div>

## Acknowledgments

I would like to thank [Dan Ehrenberg](https://twitter.com/littledan) for tireless championing of the class features and help editing this article, and [Rob Palmer](https://twitter.com/robpalmer2) for proofing and suggestions.

## Edits

1. <span class="onum">2 May, 2018</span> &mdash; Corrected example 38; reformatted to read better on mobile.
1. <span class="onum">3 May, 2018</span> &mdash; Added `this` semantics to example 39.
1. <span class="onum">4 May, 2018</span> &mdash; Corrected example 3. (From [Thomas Chetwin](https://twitter.com/TChetwin))

## Specification Footnotes

<a href="#tdz-use" id="tdz">(*)</a> In subclass constructors, `this` is in the TDZ (temporal dead zone) until `super()` returns.

<a href="#homeobj-use" id="homeobj">(&dagger;)</a> In instance methods, [[HomeObject]] is the class prototype.

<a href="#homeobj-use2" id="homeobj2">(&Dagger;)</a> In static methods, [[HomeObject]] is the class constructor.

<script>
var permanchors = document.getElementsByClassName("permanchor");
for (var i = 0; i < permanchors.length; i++) {
  permanchors[i].href = "#ex" + (i+1);
  permanchors[i].id = "ex" + (i+1);
}
</script>
