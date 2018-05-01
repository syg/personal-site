---
layout: post
title: The Semantics of All Class Elements
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

## Mental Models

Here are the mental models for the columns above. These intuitions will be bolstered by a deep dive into the semantics with concrete examples in the next section.

### Visibility

A **public** element is a writable and configurable property. As a property, public things participate in prototype inheritance.

A **private** element is that which is accessible via lexically scoped private names only on objects of distinguished provenance. Since they are not properties, they also do not participate in prototype inheritance.

Thought of another way, the behavior of private is a strict subset of the behavior of public. Encapsulation is enabled precisely by combination of lexical scoping with the provenance restriction on object dispatch. The JS concept of private is designed for JS and differs from "private" in other languages.

### Placement

An **instance** element is accessed on instances of a class.

A **static** element is accessed on class constructors.

### Class Element

A **field** is state associated with a class that has a blessed declarative syntax. An instance field is per-instance state. A static field is per-class constructor state.

A **method** is some behavior associated with a class that has a blessed declarative syntax. Both instance and static method have one identity per-class evaluation. Instance methods are on the prototype, and static methods are on the class constructor.

## Let's Make Semantics Concrete with Examples

I hope to make clear the semantics of all class elements below with examples. Each example will be followed by a short explanation of the highlighted semantics.

### Public instance fields

Classes may declare public fields accessible as properties on instances.

Public instance fields are properties added by [[DefineOwnProperty]]. They are added at [[Construct]] time of the object for the base class, before the constructor body runs.

{% highlight js %}
class C {
  publicField;

  constructor() {
    let desc = Object.getOwnPropertyDescriptor(this, "publicField");
    assert(desc.value === undefined);
    assert(desc.writable);
    assert(desc.enumerable);
    assert(desc.configurable);
  }
}

let c = new C;
{% endhighlight %}

* * *

For subclasses, `this` is in TDZ (throws `ReferenceError` if touched) until `super()` is called. So, public instance fields are added when `super()` returns.

{% highlight js %}
class Base {
  basePublicField;
}

class Sub extends Base {
  subPublicField;

  constructor() {
    super();

    assert(this.hasOwnProperty("basePublicField"));
    assert(this.hasOwnProperty("subPublicField"));
  }
}

let s = new Sub;
{% endhighlight %}

* * *

All constructors in JavaScript can return a different object, overriding the result from `new` and the newly bound `this` value from `super()`. For instance fields, if the super constructor returns something different, fields from the subclass are still added.

{% highlight js %}
class Base {
  basePublicField;
}

class ReturnTrickBase {
  trickyBasePublicField;

  constructor() {
    return new Base;
  }
}

class ReturnTrickSub extends ReturnTrickBase {
  trickySubPublicField;

  constructor() {
    super();

    assert(!this.hasOwnProperty("trickyBasePublicField"));
    assert(this.hasOwnProperty("basePublicField"));
    assert(this.hasOwnProperty("trickySubPublicField"));
  }
}
{% endhighlight %}

* * *

Public field names may be computed, like properties. They are evaluated once per class evaluation.

{% highlight js %}
let count = 0;
function makeC(sym) {
  return class C {
    [(count++, sym)];
  };
}

let key = Symbol("key");
let C = makeC(key);
assert(count === 0);
let c1 = new C;
assert(c1.hasOwnProperty(key), true);
assert(count === 1);
let c2 = new C;
assert(c2.hasOwnProperty(key), true);
assert(count === 1);
{% endhighlight %}

### Public instance methods

Classes may declare public methods accessible on the instances via the prototype.

Public instance methods are added to the class prototype with [[DefineOwnProperty]] at class evaluation time. They are writable, non-enumerable, and configurable.

{% highlight js %}
class C {
  publicMethod() { return 42; }
}

let desc = Object.getOwnPropertyDescriptor(C.prototype, "publicMethod");
assert(desc.value === C.prototype.publicMethod);
assert(desc.writable);
assert(!desc.enumerable);
assert(desc.configurable);
{% endhighlight %}

* * *

Generator function, async function, and async generator function forms may also be public instance methods.

{% highlight js %}
class C {
  *publicGeneratorMethod() { }
  async publicAsyncMethod() { }
  async *publicAsyncGeneratorMethod() { }
}
{% endhighlight %}

* * *

Getters and setters are possible as well. There are no generator, async, or async generator getter and setter forms.

{% highlight js %}
class C {
  get accessorPublicField() { return 42; }
  set accessorPublicField(x) { }
}

let desc = Object.getOwnPropertyDescriptor(C.prototype, "accessorPublicField");
assert(desc.value === undefined);
assert(!desc.enumerable);
assert(desc.configurable);
{% endhighlight %}

* * *

[[HomeObject]] is the class prototype in instance methods. `super` gets the [[Prototype]] value of the [[HomeObject]], and in this case invokes `Base.prototype.basePublicMethod`.

{% highlight js %}
class Base {
  basePublicMethod() { return 42; }
}

class Sub extends Base {
  subPublicMethod() {
    return super.basePublicMethod();
  }
}

let s = new Sub;
assert(s.subPublicMethod() === 42);
{% endhighlight %}

### Private instance fields

Classes may declare private fields accessible on base class or subclass instances from inside the lexical scope of the class declaration itself.

Instance fields that are declared with Private Names, identifiers that are prefixed with `##`, are private instance fields. `##` is the new `_`, with encapsulation being enforced by the language instead of convention. `##` is part of the name itself and is used in both in declaration and access.

{% highlight js %}
class C {
  #privateField;

  constructor() {
    this.#privateField = 42;
  }
}
{% endhighlight %}

* * *

The lexical scoping rules of Private Names are stricter than that of properties. It is an Early Error to refer to Private Names that are not in scope.

{% highlight js %}
class C {
  #privateField;

  constructor() {
    this.##nonExistentField = 42; // Syntax Error
  }
}
{% endhighlight %}
{% highlight js %}
// Syntax Error
class C {
  #privateField;
}
let c = new C;
c.#privateField; // Syntax Error
{% endhighlight %}

* * *

It is also an Early Error to `delete` Private Names.

{% highlight js %}
class C {
  #privateField;
  constructor() {
    delete this.#privateField; // Syntax Error
  }
}
{% endhighlight %}

* * *

Private field accesses are strictly more restrictive than public field accesses. Getting a Private Name on an object that doesn't have it is a `TypeError`. Attempting to set a Private Name on an object that doesn't have it is also a `TypeError`. Private fields are not properties, and do not participate in the machinery of property lookup like prototype inheritance. This is to enable encapsulation, as property lookup machinery is observable, e.g., by `Proxy`.

Private fields combines lexical scoping with a provenance restriction on dispatch. For private instance fields, the provenance is having been constructed, either as a base class or a subclass, by the class that declared the private instance field.

{% highlight js %}
class C {
  #privateField;

  getField() { return this.#privateField; }
  setField(v) { this.#privateField = v; }
}

let c = new C;
assertThrows(() => c.getField.call({}), TypeError);
assertThrows(() => c.setField.call({}, 42), TypeError);
assertThrows(() => c.getField.call(Object.create(c)), TypeError);
{% endhighlight %}

* * *

Private Names are accessible through direct `eval`, like other lexically scoped things.

{% highlight js %}
class C {
  #privateField;

  constructor() {
    eval("this.#privateField = 42");
  }
}

let c = new C;
{% endhighlight %}

* * *

Private fields are added at the same time as public fields, either at [[Construct]] time in the base class, or after `super()` returns in a subclass.

{% highlight js %}
class Base {
  ##basePrivateField;
}

class Sub extends Base {
  ##subPrivateField;

  constructor() {
    super();

    this.##basePrivateField = 42;
    this.##subPrivateField = 84;
  }
}

let s = new Sub;
{% endhighlight %}

* * *

Like with public instance fields, if the `super()` overrides the return value, private fields from the subclass are still added. For implementers, this means private fields may be added to arbitrary objects.

{% highlight js %}
class ReturnTrickBase {
  constructor() {
    return new Proxy({}, {});
  }
}

class ReturnTrickSub extends ReturnTrickBase {
  ##subPrivateField;

  constructor() {
    super();

    this.##subPrivateField = 42;
  }
}
{% endhighlight %}

* * *

While Private Names are not first class in the language, they have observably distinct per-class evaluation identities. One evaluation of a class declaration cannot access the private fields of another evaluation of the same declaration.

{% highlight js %}
function makeC() {
  return class C {
    #privateField;

    getField() { return this.#privateField; }
  };
}

let c1 = new makeC();
let c2 = new makeC();
assertThrows(() => c1.getField.call(c2), TypeError);
assertThrows(() => c2.getField.call(c1), TypeError);
{% endhighlight %}

Finally, note that there is currently no shorthand notation for accessing Private Names. Their access require a receiver.

### Private instance methods

Private instance methods are analogous to public instance methods. Their access is restricted in the same fashion as private instance fields.

These methods are specified as non-writable private fields of class instances. Like private instance fields, they are added in [[Construct]] for the base classes and after `super()` returns for subclasses.

{% highlight js %}
class C {
  #privateMethod() { return 42; }

  constructor() {
    assert(this.#privateMethod() === 42);
    assertThrows(() => this.#privateMethod = null, TypeError);
  }
}

let c = new C;
{% endhighlight %}

* * *

Generator function, async function, and async generator function forms may also be private instance methods.

{% highlight js %}
class C {
  *#privateGeneratorMethod() { }
  async #privateAsyncMethod() { }
  async *#privateAsyncGeneratorMethod() { }
}
{% endhighlight %}

* * *

Getters and setters are possible as well. There are no generator, async, or async generator getter and setter forms for private instance methods.

Private fields are specified as a per-instance list of map of Private Names to property descriptors. This preserves the symmetry of expressible forms with public instance methods, as well as enforce the restrictions that come with private fields.

{% highlight js %}
class C {
  get ##accessorPrivateField() { return 42; }
  set ##accessorPrivateField(x) { }

  constructor() {
    assert(this.##accessorPrivateField === 42);
    this.##accessorPrivateField = "hmm";
  }
}

let c = new C;
{% endhighlight %}

* * *

There is a single function identity per class-evaluation for private instance methods. Even though they are specified as per-instance private fields, instance methods are shared across all instances.

{% highlight js %}
let exfiltrated;
class C {
  #privateMethod() { }

  constructor() {
    if (exfiltrated == undefined) {
      exfiltrated = this.#privateMethod;
    }
    assert(exfiltrated === this.#privateMethod);
  }
}

let c = new C;
{% endhighlight %}

* * *

[[HomeObject]] and `this` in private instance methods follow the same semantics as those in public instance methods. Since private fields and methods are not added to the prototype, `#privateMethod()` throws. Similarly, when an object of incompatible provenance is passed as the receiver to an instance private method, the private field lookup throws.

{% highlight js %}
class Base {
  #privateMethod() { return 42; }
}

class Sub extends Base {
  #privateMethod() {
    assert(super === Base.prototype);
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

let s = new Sub;
assertThrows(() => s.publicMethod.call({}), TypeError);
{% endhighlight %}

### Instance field initializers

All fields can be initialized with an initializer expression in situ with the declaration. Initializers are run in declaration order. Instance field initializer expressions are specified as the bodies of non-observable instance methods, which informs the values of `this`, `new.target`, and [[HomeObject]].

Fields without initializers are initialized to `undefined`.

{% highlight js %}
class C {
  #privateField = 42;
  publicFieldOne = 84;
  publicFieldTwo;

  constructor() {
    assert(this.#privateField === 42);
    assert(this.publicFieldOne === 84);
    assert(this.publicFieldTwo === undefined);
  }
}

let c = new C;
{% endhighlight %}

* * *

Field initializers are run as fields are added, in declaration order, and this order is observable by the initializers. `#privateField` is `false` in the above as `publicField` has has not been added yet when evaluating `#privateField`'s initializer. There is also no error when initializing `publicField` with `this.#privateField`, owing to its coming after `#privateField`.

These initializers are specified as methods that return the result of the initializer expressions. The methods need not be reified in implementation, but this fiction of specification informs the values of `this`. In instance field initializers, `this` is the object under construction. By the time the instance field initializer runs, `this` is out of TDZ.

{% highlight js %}
class C {
  #privateField = this.hasOwnProperty("publicField");
  publicField = this.#privateField;

  constructor() {
    assert(!this.#privateField);
    assert(!this.publicField);
  }
}

let c = new C;
{% endhighlight %}

* * *

Instance methods are added before any initializer is run, so all instance methods are available in instance field initializers.

{% highlight js %}
class C {
  #privateField = this.#privateMethod();
  publicField = this.#privateField;

  #privateMethod() { return 42; }
}

let c = new C;
assert(c.publicField === 42);
{% endhighlight %}

* * *

`new.target` is `undefined` in field initializers.

{% highlight js %}
class C {
  publicField = new.target;
}

let c = new C;
assert(c.publicField === undefined);
{% endhighlight %}

* * *

[[HomeObject]] is the class's prototype in instance field initializers.

{% highlight js %}
class Base {
  baseMethod() { return 42; }
}

class Sub extends Base {
  subPublicField = super.baseMethod();
}

let s = new Sub;
assert(s.subPublicField === 42);
{% endhighlight %}

* * *

Class bodies are always strict code, so initializers cannot leak any new bindings via direct `eval`.

{% highlight js %}
class C {
  publicField = eval("var x = 42;");
}
let c = new C;
assertThrows(() => x, ReferenceError);
{% endhighlight %}

* * *

Use of `arguments` is an Early Error in field initializers.

{% highlight js %}
// Syntax Error
class C {
  publicField = arguments;
}
{% endhighlight %}

### Public static fields

Classes may declare public static fields, which are accessible as properties on the class constructor.

Public static fields are added to the class constructor with [[DefineOwnProperty]] at class evaluation time. Their semantics are otherwise identical to public instance fields.

{% highlight js %}
class C {
  static STATIC_PUBLIC_FIELD;
}

let desc = Object.getOwnPropertyDescriptor(C, "STATIC_PUBLIC_FIELD");
assert(desc.value === undefined);
assert(desc.writable);
assert(desc.enumerable);
assert(desc.configurable);
{% endhighlight %}

* * *

Public static fields are only initialized on the class in which they are defined, not reinitialized on subclasses. Subclass constructors have their superclasses as the value of [[Prototype]]. Public static fields of superclasses are accessed via the prototype chain.

{% highlight js %}
class Base {
  static BASE_STATIC_PUBLIC_FIELD;
}

class Sub extends Base {
  static SUB_STATIC_PUBLIC_FIELD;
}

assert(Base.hasOwnProperty("BASE_STATIC_PUBLIC_FIELD"));
assert(Sub.hasOwnProperty("SUB_STATIC_PUBLIC_FIELD"));
assert(Object.getPrototypeOf(Sub) === Base);
{% endhighlight %}

### Public static methods

Public static methods are declared function forms and, like public static fields, are also accessible as properties on the class constructor.

These methods are added to the class constructor with [[DefineProperty]] at class evaluation time. Like public instance methods, they are writable, non-enumerable, and configurable.

Also like public instance methods, generator function, async function, async generator function, getter, and setter forms are accepted.

{% highlight js %}
class C {
  static staticPublicMethod() { return 42; }
}

let desc = Object.getOwnPropertyDescriptor(C, "staticPublicMethod");
assert(desc.value === C.publicMethod);
assert(desc.writable);
assert(!desc.enumerable);
assert(desc.configurable);
{% endhighlight %}

* * *

[[HomeObject]] is the class constructor in static methods. As subclass constructors have as their [[Prototype]] their superclass constructor, the super class's public static methods are accessible with `super`.

{% highlight js %}
class Base {
  static baseStaticPublicMethod() { return 42; }
}

class Sub extends Base {
  static subStaticPublicMethod() {
    return super.baseStaticPublicMethod();
  }
}
{% endhighlight %}

### Private static fields

Classes may declare private fields accessible on the class constructor from inside the lexical scope of the class declaration itself.

Private static fields are added to the class constructor at class evaluation time.

{% highlight js %}
class C {
  static #STATIC_PRIVATE_FIELD;

  static staticPublicMethod() {
    C.#STATIC_PRIVATE_FIELD = 42;
    return C.#STATIC_PRIVATE_FIELD;
  }
}

assert(C.staticPublicMethod() === 42);
{% endhighlight %}

* * *

The provenance restriction of static Private Names is the class constructor of the class in which they are defined. The above throws because the `this` value of `baseStaticPublicMethod` is the subclass constructor, which does not have the `#BASE_STATIC_PRIVATE_FIELD` field. This is the natural result, though unexpected for some, of the composition of the Private Name provenance restriction with `this` dynamicity.

{% highlight js %}
class Base {
  static #BASE_STATIC_PRIVATE_FIELD;

  static baseStaticPublicMethod() {
    return this.#BASE_STATIC_PRIVATE_FIELD;
  }
}

class Sub extends Base { }

assertThrows(() => Sub.baseStaticPublicMethod(), TypeError);
{% endhighlight %}

### Private static methods

Private static methods are analogous to public static methods. Their access is restricted in the same fashion as private static fields.

These methods are specified as non-writable private fields of class constructors. Like private static fields, they are added at class evaluation time.

Like all methods, generator function, async function, async generator function, getter, and setter forms are accepted.

{% highlight js %}
class C {
  static #staticPrivateMethod() { return 42; }

  constructor() {
    assertThrows(C.#staticPrivateMethod = null, TypeError);
    assert(C.#staticPrivateMethod(), 42);
  }
}

let c = new C;
{% endhighlight %}

* * *

Like public static methods, [[HomeObject]] is the class constructor.

{% highlight js %}
class Base {
  static baseStaticPublicMethod() { return 42; }
}

class Sub extends Base {
  static #subStaticPrivateMethod() {
    assert(super.baseStaticPublicMethod() === 42);
  }
}
{% endhighlight %}

### Static field initializers

Static fields may have in-situ initializers. Like instance field initializers, they are also run in declaration order, and the expressions are specified as bodies of non-observable static methods.

As initializers expressions are specified as static method bodies, [[HomeObject]] is the class constructor.

{% highlight js %}
class Base {
  static baseStaticPublicMethod() { return 42; }
}

class Sub extends Base {
  static SUB_STATIC_PUBLIC_FIELD = super.baseStaticMethod();
}

assert(Sub.SUB_STATIC_PUBLIC_FIELD === 42);
{% endhighlight %}

* * *

By the time static field initializers are evaluated, the class name binding inside the class scope is initialized (i.e. out of TDZ).

{% highlight js %}
class C {
  static #STATIC_PRIVATE_FIELD = 42;
  static STATIC_PUBLIC_FIELD = C.#STATIC_PRIVATE_FIELD;
}

assert(C.STATIC_PUBLIC_FIELD === 42);
{% endhighlight %}

## Some Take Aways

We've explored the semantics of all JavaScript class elements. Perhaps some semantics were surprising, while others were expected. I believe a common source of mismatched intuition is the provenance restriction on Private Names, and I hope this article was helpful in making this clearer.
