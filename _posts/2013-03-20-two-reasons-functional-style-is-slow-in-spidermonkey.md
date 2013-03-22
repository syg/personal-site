---
layout: post
title: Two Reasons Functional Style Is Slow in SpiderMonkey
---

{{ page.title }}
================

Programming JS in a functional style is often more elegant but comes at the
cost of performance. In the past, part of that performance problem has been
due to native, C++ implementations of the most-often used higher-order
operations like `Array.prototype.forEach` and friends. In a call like
`arr.forEach(f)`, the passed in kernel function `f` ends up needing to
cross the C++-JS boundary every iteration of the `forEach`, causing
slowdown. Moreover, the JIT cannot be leveraged as `forEach` is not a JS
function, and thus cannot be analyzed and optimized.

With the [hard work](https://bugzilla.mozilla.org/show_bug.cgi?id=784294) of
[Till Schneidereit](http://www.tillschneidereit.de/), SpiderMonkey now has
self-hosted versions of these higher-order operations. That is, the
implementation is in JS, not C++. This gets rid of the C++-JS boundary, and in
theory, should leverage the JIT to get much improved performance. So, let's
write a simple benchmark that finds the maximum value in a N&#215;N matrix
represented raggedly using vanilla JS arrays, adopted from
[here](http://jsperf.com/scan-max-index/3).

**Note**: The following benchmarks should be run on Firefox Nightly. My
comments about what's fast and what's slow probably (definitely) do not apply
to other browsers and versions.

As a preamble, a utility benchmark function that creates a N&#215;N matrix of
random numbers, takes a benchmark function `f`, and runs it `iters` times:

{% highlight js %}
function benchmark(n, iters, f) {
  var outer = [];
  for (var i = 0; i < n; i++) {
    var inner = [];
    for (var j = 0; j < n; j++)
      inner.push(Math.random() * 100)
    outer.push(inner);
  }
  var start = Date.now();
  for (var i = 0; i < iters; i++)
    f(outer);
  return Date.now() - start;
}
{% endhighlight %}

First is the gold standard, two nested `for` loops. This performs quite well.

{% highlight js %}
function forLoop(outer) {
  var max = -Infinity;
  for (var i = 0; i < outer.length; i++) {
    var inner = outer[i];
    for (var j = 0; j < inner.length; j++) {
      var v = inner[j];
      if (v > max)
        max = v;
    }
  }
}
benchmark(50, 50000, forLoop);
{% endhighlight %}

<div class="measurement">
  <span class="button" id="forLoop">run</span>
  <span class="output" id="forLoop:time"></span>
</div>

* * *

Next up is using the builtin `Array.prototype.forEach`. The functional
programmer in me rejoices, but this version performs much, much worse than the
above code.

{% highlight js %}
function arrayForEach(outer) {
  var max = -Infinity;
  outer.forEach(function (inner) {
    inner.forEach(function (v) {
      if (v > max)
        max = v;
    });
  });
}
benchmark(50, 50000, arrayForEach);
{% endhighlight %}

<div class="measurement">
  <span class="button" id="arrayForEach">run</span>
  <span class="output" id="arrayForEach:time"></span>
</div>

* * *

To convince yourself that a handrolled `forEach` does no better, run the
following snippet. Note however that a handrolled version is a bit faster, due
to its not needing to perform extra operations that the builtin version is
required to perform for spec compliance.

{% highlight js %}
Array.prototype.myForEach = function (f) {
  for (var i = 0; i < this.length; i++)
    f(this[i]);
}
function myForEach(outer) {
  max = -Infinity;
  outer.myForEach(function (inner) {
    inner.myForEach(function (v) {
      if (v > max)
        max = v;
    });
  });
}
benchmark(50, 50000, myForEach);
{% endhighlight %}

<div class="measurement">
  <span class="button" id="myForEach">run</span>
  <span class="output" id="myForEach:time"></span>
</div>

* * *

The performance degradation, even with a self-hosted `forEach`, is due to the
JIT's inability to efficiently inline both the closures passed to `forEach` as
well as the nested call to `forEach`. In a perfect world, the `forEach`
version, after inlining all the way, should be more or less equivalent to the
`for` loop version.

An old technique exists to compile closures more efficiently--[lambda
lifting](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.48.4346). The
optimization is currently not implemented in IonMonkey, the new JIT of
SpiderMonkey, but it's easy to verify its effectiveness by manually optimizing
the source.

The transformation is straightforward. Convert all free variables to
arguments, thus obviating the need to create an environment, then lift the
closures to the global scope. Below I choose to collect all free variables
(though there's only one in both kernels anyways) into a state object, `st`.

{% highlight js %}
Array.prototype.liftedForEach = function (f, st) {
  for (var i = 0; i < this.length; i++)
    f(this[i], st);
}
function liftedInnerKernel(v, st) {
  if (v > st.max)
    st.max = v;
}
function liftedOuterKernel(inner, st) {
  inner.liftedForEach(liftedInnerKernel, st);
}
function liftedForEach(outer) {
  var st = { max: -Infinity };
  outer.liftedForEach(liftedOuterKernel, st);
}
benchmark(50, 50000, liftedForEach);
{% endhighlight %}

<div class="measurement">
  <span class="button" id="liftedForEach">run</span>
  <span class="output" id="liftedForEach:time"></span>
</div>

* * *

The lambda lifted version is faster, but still not up to the speed of the
nested `for` loops. What's going on? It turns out that the call to `f` inside
`liftedForEach` is unable to be inlined effectively due to its being
polymorphic; both `liftedInnerKernel` and `liftedOuterKernel` are passed to
`f`. We can help the compiler by manually cloning the caller.

{% highlight js %}
Array.prototype.liftedForEach1 = function (f, st) {
  for (var i = 0; i < this.length; i++)
    f(this[i], st);
}
Array.prototype.liftedForEach2 = function (f, st) {
  for (var i = 0; i < this.length; i++)
    f(this[i], st);
}
function liftedInnerKernel(v, st) {
  if (v > st.max)
    st.max = v;
}
function liftedAndClonedOuterKernel(inner, st) {
  inner.liftedForEach2(liftedInnerKernel, st);
}
function liftedAndClonedForEach(outer) {
  var st = { max: -Infinity };
  outer.liftedForEach1(liftedAndClonedOuterKernel, st);
}
benchmark(50, 50000, liftedAndClonedForEach);
{% endhighlight %}

<div class="measurement">
  <span class="button" id="liftedAndClonedForEach">run</span>
  <span class="output" id="liftedAndClonedForEach:time"></span>
</div>

* * *

Finally, comparable performance to the nested `for` loops. To conclude, there
are two criteria to functional style performance parity:

1. Efficient compilation and inlining of closures.
2. Enough context sensitivity to obtain monomorphic callsites.

The transformations to enable the above criteria are tedious and are surely
the purview of the compiler. All that's needed are brave compiler hackers.
<img id="seal" src="{{ site.baseurl }}/seal.svg" alt="seal">

<script>
function benchmark(n, iters, f) {
  var outer = [];
  for (var i = 0; i < n; i++) {
    var inner = [];
    for (var j = 0; j < n; j++)
      inner.push(Math.random() * 100)
    outer.push(inner);
  }
  var start = Date.now();
  for (var i = 0; i < iters; i++)
    f(outer);
  return Date.now() - start;
}

var benchs = [];

benchs["forLoop"] = function forLoop(outer) {
  var max = -Infinity;
  for (var i = 0; i < outer.length; i++) {
    var inner = outer[i];
    for (var j = 0; j < inner.length; j++) {
      var v = inner[j];
      if (v > max)
        max = v;
    }
  }
};

benchs["arrayForEach"] = function arrayForEach(outer) {
  var max = -Infinity;
  outer.forEach(function (inner) {
    inner.forEach(function (v) {
      if (v > max)
        max = v;
    });
  });
};

Array.prototype.myForEach = function (f) {
  for (var i = 0; i < this.length; i++)
    f(this[i]);
}

benchs["myForEach"] = function myForEach(outer) {
  max = -Infinity;
  outer.myForEach(function (inner) {
    inner.myForEach(function (v) {
      if (v > max)
        max = v;
    });
  });
};

Array.prototype.liftedForEach = function (f, st) {
  for (var i = 0; i < this.length; i++)
    f(this[i], st);
}
function liftedInnerKernel(v, st) {
  if (v > st.max)
    st.max = v;
}
function liftedOuterKernel(inner, st) {
  inner.liftedForEach(liftedInnerKernel, st);
}

benchs["liftedForEach"] = function liftedForEach(outer) {
  var st = { max: -Infinity };
  outer.liftedForEach(liftedOuterKernel, st);
};

Array.prototype.liftedForEach1 = function (f, st) {
  for (var i = 0; i < this.length; i++)
    f(this[i], st);
}
Array.prototype.liftedForEach2 = function (f, st) {
  for (var i = 0; i < this.length; i++)
    f(this[i], st);
}
function liftedAndClonedOuterKernel(inner, st) {
  inner.liftedForEach2(liftedInnerKernel, st);
}

benchs["liftedAndClonedForEach"] = function liftedAndClonedForEach(outer) {
  var st = { max: -Infinity };
  outer.liftedForEach1(liftedAndClonedOuterKernel, st);
};

var runButtons = document.getElementsByClassName("button");
for (var i = 0; i < runButtons.length; i++) {
  var button = runButtons[i];
  button.onclick = function() {
     var time = benchmark(50, 50000, benchs[this.id]);
     document.getElementById(this.id + ":time").innerHTML = time + " ms";
  };
}
</script>
