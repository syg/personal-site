---
layout: post
title: The Semantics of All JS Class Elements
---

{{ page.title }}
================

This article summarizes the current and proposed class fields and methods semantics. With the exception of static private fields and methods at the time of writing, all the described semantics enjoy current <span class="nnum">TC39</span> consensus.

The table of proposed features is as follows. The full feature set is the product of the columns.

| Visibility | Placement | Class Element |
|------------|-----------|---------------|
| Public     | Instance  | Field         |
| Private    | Static    | Method        |

I will describe the semantics of each column in detail, as well as the consequences of their combinations with each other as and with existing JS language features.

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

A **method** is some behavior associated with a class that has a blessed declarative syntax. Both instance and static method have one identity per-class evaluation. Instance methods are on the prototype, and static methods are on the class constructor.

## Let's Make the Semantics Concrete with Examples

I hope to make clear the semantics of all class elements below with examples. Each example will be followed by a short explanation of the highlighted semantics.

### Public instance fields

Classes may declare public fields accessible as properties on instances.

<a class="permanchor"></a>
Public instance fields are properties added by `Object.defineProperty`. They are added at construction time of the object for the base class, before the constructor body runs.

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

* * *

<a class="permanchor"></a>
For subclasses, `this` throws `ReferenceError` if touched until `super()` is called.<sup><a href="#tdz" id="tdz-use">(*)</a></sup> So, public instance fields are added when `super()` returns.

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

* * *

<a class="permanchor"></a>
All constructors in JavaScript can return a different object, overriding the result from `new` and the newly bound `this` value from `super()`. For instance fields, if the super constructor returns something different, fields from the subclass are still added.

{% highlight js %}
class Ex3_Base {
  basePublicField;
}

class Ex3_ReturnTrickBase {
  trickyBasePublicField;

  constructor() {
    return new Base;
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

* * *

<a class="permanchor"></a>
Public field names may be computed, like properties. They are evaluated once per class evaluation.

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
assert(ex4a.hasOwnProperty(key), true);
assert(count === 1);
let ex4b = new Ex4;
assert(ex4b.hasOwnProperty(key), true);
assert(count === 1);
{% endhighlight %}

### Public instance methods

Classes may declare public methods accessible on the instances via the prototype.

<a class="permanchor"></a>
Public instance methods are added to the class prototype with `Object.defineProperty` at class evaluation time. They are writable, non-enumerable, and configurable.

{% highlight js %}
class Ex5 {
  publicMethod() { return 42; }
}

let desc = Object.getOwnPropertyDescriptor(Ex5.prototype, "publicMethod");
assert(desc.value === Ex5.prototype.publicMethod);
assert(desc.writable);
assert(!desc.enumerable);
assert(desc.configurable);
{% endhighlight %}

* * *

<a class="permanchor"></a>
Generator function, async function, and async generator function forms may also be public instance methods.

{% highlight js %}
class Ex6 {
  *publicGeneratorMethod() { }
  async publicAsyncMethod() { }
  async *publicAsyncGeneratorMethod() { }
}
{% endhighlight %}

* * *

<a class="permanchor"></a>
Getters and setters are possible as well. There are no generator, async, or async generator getter and setter forms.

{% highlight js %}
class Ex7 {
  get accessorPublicField() { return 42; }
  set accessorPublicField(x) { }
}

let desc = Object.getOwnPropertyDescriptor(Ex7.prototype, "accessorPublicField");
assert(desc.value === undefined);
assert(!desc.enumerable);
assert(desc.configurable);
{% endhighlight %}

* * *

<a class="permanchor"></a>
In instance methods, `super` references the superclass's `prototype` property. The following invokes `Base.prototype.basePublicMethod`.<sup><a href="#homeobj" id="homeobj-use">(&dagger;)</a></sup>

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

### Private instance fields

Classes may declare private fields accessible on base class or subclass instances from inside the lexical scope of the class declaration itself.

<a class="permanchor"></a>
Private instance fields are declared with **<span style="font-family: bold 16px 'Cousine', monospace">#</span> names** (said "hash names"), identifiers that are prefixed with `#`. Though a different character, this follows the convention of signaling privacy with `_`-prefixed property names.

<span style="font-variant: small-caps; font-size: 200%; float: left; margin-right: 0.4em; line-height: 48px"><span style="font: bold 75% 'Cousine', monospace">#</span> is the new <span style="font: bold 75% 'Cousine', monospace">_</span></span>with encapsulation being enforced by the language instead of by convention. `#` is part of the name itself and is used in both in declaration and access.

{% highlight js %}
class Ex9 {
  #privateField;

  constructor() {
    this.#privateField = 42;
  }
}

new Ex9;
{% endhighlight %}

* * *

<a class="permanchor"></a>
The lexical scoping rules of `#` names are stricter than those of identifier names. It is a syntax error to refer to `#` names that are not in scope.

{% highlight js %}
class Ex10_A {
  #privateField;

  constructor() {
    this.#nonExistentField = 42; // Syntax error
  }
}
{% endhighlight %}
{% highlight js %}
// Syntax error
class Ex10_B {
  #privateField;
}
(new Ex10_B).#privateField; // Syntax error
{% endhighlight %}

* * *

<a class="permanchor"></a>
Like lexical bindings, it is a syntax error to have multiple same-named `#` names in the same scope (i.e. the class declaration), while shadowing via nested scopes is allowed.

Note that since all `#` names start with `#` and property names cannot start with `#`, the two cannot be in conflict.

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

    assert((new Ex11_Inner).privateFieldValue() == 42);
    assert(this.#privateField === undefined);
  }
}

new Ex11_Outer;
{% endhighlight %}

* * *

<a class="permanchor"></a>
It is also a syntax error to `delete` `#` names.

{% highlight js %}
class Ex12 {
  #privateField;
  constructor() {
    delete this.#privateField; // Syntax error
  }
}
{% endhighlight %}

* * *

<a class="permanchor"></a>
Private field accesses are strictly more restrictive than public field accesses. Getting a `#` name on an object that doesn't have a private field with that name is a `TypeError`. The same goes for setting a `#` name. Private fields are not properties, and do not participate in the machinery of property lookup like prototype inheritance. This is to enable encapsulation, as property lookup machinery is observable, e.g., by `Proxy`.

Private fields combines lexical scoping with a provenance restriction on dispatch. For private instance fields, the provenance is having been constructed, either as a base class or a subclass, by the class that declared the private instance field.

{% highlight js %}
class Ex13 {
  #privateField;

  getField() { return this.#privateField; }
  setField(v) { this.#privateField = v; }
}

let ex13 = new Ex13;
assertThrows(() => ex13.getField.call({}), TypeError);
assertThrows(() => ex13.setField.call({}, 42), TypeError);
assertThrows(() => ex13.getField.call(Object.create(c)), TypeError);
{% endhighlight %}

* * *

<a class="permanchor"></a>
`#` names are accessible through direct `eval`, like other lexically scoped things.

{% highlight js %}
class Ex14 {
  #privateField;

  constructor() {
    eval("this.#privateField = 42");
  }
}

new Ex14;
{% endhighlight %}

* * *

<a class="permanchor"></a>
Private fields are added at the same time as public fields, either at construction time in the base class, or after `super()` returns in a subclass.

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

* * *

<a class="permanchor"></a>
Like with public instance fields, if the `super()` overrides the return value, private fields from the subclass are still added. For implementers, this means private fields may be added to arbitrary objects.

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

* * *

<a class="permanchor"></a>
While `#` names are not first class in the language, they have observably distinct per-class evaluation identities. One evaluation of a class declaration cannot access the private fields of another evaluation of the same declaration.

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

Finally, note that there is currently no shorthand notation for accessing `#` names. Their access require a receiver.

### Private instance methods

Private instance methods are analogous to public instance methods. Their access is restricted in the same fashion as private instance fields.

<a class="permanchor"></a>
These methods are specified as non-writable private fields of class instances. Like private instance fields, they are added at construction time for the base classes and after `super()` returns for subclasses.

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

* * *

<a class="permanchor"></a>
Generator function, async function, and async generator function forms may also be private instance methods.

{% highlight js %}
class Ex19 {
  *#privateGeneratorMethod() { }
  async #privateAsyncMethod() { }
  async *#privateAsyncGeneratorMethod() { }
}
{% endhighlight %}

* * *

<a class="permanchor"></a>
Getters and setters are possible as well. There are no generator, async, or async generator getter and setter forms for private instance methods.

Private fields are specified as a per-instance list of map of `#` names to property descriptors. This preserves the symmetry of expressible forms with public instance methods, as well as enforce the restrictions that come with private fields.

{% highlight js %}
class Ex20 {
  get #accessorPrivateField() { return 42; }
  set #accessorPrivateField(x) { }

  constructor() {
    assert(this.#accessorPrivateField === 42);
    this.#accessorPrivateField = "hmm";
  }
}

new Ex20;
{% endhighlight %}

* * *

<a class="permanchor"></a>
There is a single function identity per class-evaluation for private instance methods. Even though they are specified as per-instance private fields, instance methods are shared across all instances.

{% highlight js %}
let exfiltrated;
class Ex21 {
  #privateMethod() { }

  constructor() {
    if (exfiltrated == undefined) {
      exfiltrated = this.#privateMethod;
    }
    assert(exfiltrated === this.#privateMethod);
  }
}

new Ex21;
{% endhighlight %}

* * *

<a class="permanchor"></a>
In private instance methods, `super` and `this` follow the same semantics as those in public instance methods. Since private fields and methods are not added to the prototype, `#privateMethod()` throws below. Similarly, when an object of incorrect provenance is passed as the receiver to an instance private method, the private field lookup throws.

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

let ex22 = new Ex22_Sub;
assertThrows(() => ex22.publicMethod.call({}), TypeError);
{% endhighlight %}

### Instance field initializers

All fields can be initialized with an initializer expression in situ with the declaration. Initializers are run in declaration order. Instance field initializer expressions are specified as the bodies of non-observable instance methods, which informs the values of `this`, `new.target`, and `super`.

<a class="permanchor"></a>
Fields without initializers are initialized to `undefined`.

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

* * *

<a class="permanchor"></a>
Field initializers are run as fields are added, in declaration order, and this order is observable by the initializers. `#privateField` is `false` in the following as `publicField` has has not been added yet when evaluating `#privateField`'s initializer. There is also no error when initializing `publicField` with `this.#privateField`, owing to its coming after `#privateField`.

These initializers are specified as methods that return the result of the initializer expressions. The methods need not be reified in implementation, but this fiction of specification informs the values of `this`. In instance field initializers, `this` is the object under construction. By the time the instance field initializer runs, `this` is accessible.

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

* * *

<a class="permanchor"></a>
Multiple same-named public fields and methods are allowed in the same class declaration, and their initializers are run in order. Since public methods and fields are properties added with `Object.defineProperty`, the last field or method overrides all previous ones.

This does not happen with private fields and methods, as multiple same-named `#` names in the same scope is a syntax error.

{% highlight js %}
let log = ""
class Ex25 {
  publicField = (log += "1", 42);
  publicField = (log += "2", 84);
  publicField() { }
}

let ex25 = new Ex25;
assert(typeof ex25.publicField === "function");
assert(log === "12");
{% endhighlight %}

* * *

<a class="permanchor"></a>
Instance methods are added before any initializer is run, so all instance methods are available in instance field initializers.

{% highlight js %}
class Ex26 {
  #privateField = this.#privateMethod();
  publicField = this.#privateField;

  #privateMethod() { return 42; }
}

assert((new Ex26).publicField === 42);
{% endhighlight %}

* * *

<a class="permanchor"></a>
`new.target` is `undefined` in field initializers.

{% highlight js %}
class Ex27 {
  publicField = new.target;
}

assert((new Ex27).publicField === undefined);
{% endhighlight %}

* * *

<a class="permanchor"></a>
In static methods, `super` references the superclass's prototype in instance field initializers.

{% highlight js %}
class Ex28_Base {
  baseMethod() { return 42; }
}

class Ex28_Sub extends Ex28_Base {
  subPublicField = super.baseMethod();
}

assert((new Ex28_Sub).subPublicField === 42);
{% endhighlight %}

* * *

<a class="permanchor"></a>
Class bodies are always strict code, so initializers cannot leak any new bindings via direct `eval`.

{% highlight js %}
class Ex29 {
  publicField = eval("var x = 42;");
}

new Ex29;
assertThrows(() => x, ReferenceError);
{% endhighlight %}

* * *

<a class="permanchor"></a>
Use of `arguments` is a syntax error in field initializers.

{% highlight js %}
// Syntax error
class Ex30 {
  publicField = arguments;
}
{% endhighlight %}

### Public static fields

Classes may declare public static fields, which are accessible as properties on the class constructor.

<a class="permanchor"></a>
Public static fields are added to the class constructor with `Object.defineProperty` at class evaluation time. Their semantics are otherwise identical to public instance fields.

{% highlight js %}
class Ex31 {
  static STATIC_PUBLIC_FIELD;
}

let desc = Object.getOwnPropertyDescriptor(Ex31, "STATIC_PUBLIC_FIELD");
assert(desc.value === undefined);
assert(desc.writable);
assert(desc.enumerable);
assert(desc.configurable);
{% endhighlight %}

* * *

<a class="permanchor"></a>
Public static fields are only initialized on the class in which they are defined, not reinitialized on subclasses. Subclass constructors have their superclasses as their prototype. Public static fields of superclasses are accessed via the prototype chain.

{% highlight js %}
class Ex32_Base {
  static BASE_STATIC_PUBLIC_FIELD;
}

class Ex32_Sub extends Ex32_Base {
  static SUB_STATIC_PUBLIC_FIELD;
}

assert(Ex32_Base.hasOwnProperty("BASE_STATIC_PUBLIC_FIELD"));
assert(Ex32_Sub.hasOwnProperty("SUB_STATIC_PUBLIC_FIELD"));
assert(Object.getPrototypeOf(Ex32_Sub) === Ex32_Base);
{% endhighlight %}

### Public static methods

Public static methods are declared function forms and, like public static fields, are also accessible as properties on the class constructor.

<a class="permanchor"></a>
These methods are added to the class constructor with `Object.defineProperty` at class evaluation time. Like public instance methods, they are writable, non-enumerable, and configurable.

Also like public instance methods, generator function, async function, async generator function, getter, and setter forms are accepted.

{% highlight js %}
class Ex33 {
  static staticPublicMethod() { return 42; }
}

let desc = Object.getOwnPropertyDescriptor(Ex33, "staticPublicMethod");
assert(desc.value === Ex33.publicMethod);
assert(desc.writable);
assert(!desc.enumerable);
assert(desc.configurable);
{% endhighlight %}

* * *

<a class="permanchor"></a>
In static methods, `super` references the superclass constructor, and the superclass's public static methods are accessible.<sup><a href="#homeobj2" id="homeobj-use2">(&Dagger;)</a></sup>

{% highlight js %}
class Ex34_Base {
  static baseStaticPublicMethod() { return 42; }
}

class Ex34Sub extends Ex34_Base {
  static subStaticPublicMethod() {
    return super.baseStaticPublicMethod();
  }
}

assert(Ex34.subStaticPublicMethod() === 42);
{% endhighlight %}

### Private static fields

Classes may declare private fields accessible on the class constructor from inside the lexical scope of the class declaration itself.

<a class="permanchor"></a>
Private static fields are added to the class constructor at class evaluation time.

{% highlight js %}
class Ex35 {
  static #STATIC_PRIVATE_FIELD;

  static staticPublicMethod() {
    Ex35.#STATIC_PRIVATE_FIELD = 42;
    return Ex35.#STATIC_PRIVATE_FIELD;
  }
}

assert(Ex35.staticPublicMethod() === 42);
{% endhighlight %}

* * *

<a class="permanchor"></a>
The provenance restriction of static private fields restricts access to the class constructor. The following throws because the `this` value of `baseStaticPublicMethod` is the subclass constructor, which does not have the `#BASE_STATIC_PRIVATE_FIELD` field. This is the natural result, though unexpected for some, of the composition of the private field provenance restriction with `this` dynamicity.

{% highlight js %}
class Ex36_Base {
  static #BASE_STATIC_PRIVATE_FIELD;

  static baseStaticPublicMethod() {
    return this.#BASE_STATIC_PRIVATE_FIELD;
  }
}

class Ex36_Sub extends Ex36_Base { }

assertThrows(() => Ex36_Sub.baseStaticPublicMethod(), TypeError);
{% endhighlight %}

### Private static methods

Private static methods are analogous to public static methods. Their access is restricted in the same fashion as private static fields.

<a class="permanchor"></a>
These methods are specified as non-writable private fields of class constructors. Like private static fields, they are added at class evaluation time.

Like all methods, generator function, async function, async generator function, getter, and setter forms are accepted.

{% highlight js %}
class Ex37 {
  static #staticPrivateMethod() { return 42; }

  constructor() {
    assertThrows(Ex37.#staticPrivateMethod = null, TypeError);
    assert(Ex37.#staticPrivateMethod(), 42);
  }
}

new Ex37;
{% endhighlight %}

* * *

<a class="permanchor"></a>
Like public static methods, `super` references the superclass constructor.

{% highlight js %}
class Ex38_Base {
  static baseStaticPublicMethod() { return 42; }
}

class Ex38_Sub extends Ex38_Base {
  static #subStaticPrivateMethod() {
    assert(super.baseStaticPublicMethod() === 42);
  }
}

new Ex38_Sub;
{% endhighlight %}

### Static field initializers

Static fields may have in-situ initializers. Like instance field initializers, they are also run in declaration order, and the expressions are specified as bodies of non-observable static methods.

<a class="permanchor"></a>
As initializers expressions are specified as static method bodies, `super` also references the superclass constructor.

{% highlight js %}
class Ex39_Base {
  static baseStaticPublicMethod() { return 42; }
}

class Ex39_Sub extends Ex39_Base {
  static SUB_STATIC_PUBLIC_FIELD = super.baseStaticMethod();
}

assert(Ex39_Sub.SUB_STATIC_PUBLIC_FIELD === 42);
{% endhighlight %}

* * *

<a class="permanchor"></a>
By the time static field initializers are evaluated, the class name binding inside the class scope is initialized (i.e. may be accessed and does not throw a `ReferenceError`).

{% highlight js %}
class Ex40 {
  static #STATIC_PRIVATE_FIELD = 42;
  static STATIC_PUBLIC_FIELD = Ex40.#STATIC_PRIVATE_FIELD;
}

assert(Ex40.STATIC_PUBLIC_FIELD === 42);
{% endhighlight %}

## Some Take Aways

We've explored the semantics of all JavaScript class elements. Perhaps some semantics were surprising, while others were expected. I believe a common source of mismatched intuition for private fields is the provenance restriction on `#` names, and I hope this article was helpful in making this clearer. In closing, I offer these two aphorisms.<img id="seal" src="{{ site.baseurl }}/seal.svg" alt="seal">

<div style="text-align: center">
<span style="font-variant: small-caps; font-size: 150%;"><span style="font: bold 75% 'Cousine', monospace">#</span> is the new <span style="font: bold 75% 'Cousine', monospace">_</span></span>
<br>
and
<br>
<span style="font-variant: small-caps; font-size: 150%">private fields have a provenance restriction</span>
</div>

### Specification Footnotes

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
