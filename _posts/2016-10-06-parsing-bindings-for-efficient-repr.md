---
layout: post
title: Parsing Binding Names for Eﬃcient Representation
---

{{ page.title }}
================

This post explains how binding names are parsed for eﬃcient representation
in the SpiderMonkey virtual machine. The story itself is a narrow one about
JavaScript, but I should hope the lessons contained are more broadly
applicable to other language runtimes where parsing of the source text is
included in the perceived execution time.

To start, by "binding name" I mean the following.

> A binding name is the name bound by a declaration. In JS, there are many
> forms of declarations. To name some: `var`, `let`, `const` declarations,
> function statements, and formal parameters. For example, `x` is the binding
> name in `var` `x`.

## Representation of Bindings

The important constraint above is "eﬃcient". It is only correct to implement
the JS speciﬁcation literally by keeping bindings in a list of maps from
strings to values. Such an implementation is ineﬃcient in both time and
space. It is slow to look up a name dynamically every time it is needed, and
it is wasteful to keep around values that the program will never need again.

We aspire to optimize bindings along both the space and time axes. To optimize
for space, we make the distinction between bindings that are live beyond the
execution of their scope and bindings that are not and represent them
differently. To optimize for time, we make the distinction between identiﬁers
that always refer to the same binding and identiﬁers that do not and access
them differently. The former arises from use of closures. When an inner
function references a binding in an outer scope, we say that binding is
_closed over_. The latter arises from the dynamicity of `eval` and
`with`. When `eval` code references a binding in an outer environment, we call
the access _dynamic_.

For example, in the following program, `var` `x` is closed over by `g`:

{% highlight js %}
function f() {
  var x;
  function g() { x = 42; }
}
{% endhighlight %}

In the following program, it is unknown if `eval(unknownInputStr)` will close
over `x`. For example, `unknownInputStr` may be `"function g() { x = 42;
}"`. For correctness, compilers must conservatively assume that it will.

{% highlight js %}
function f(str) {
  var x = 42;
  eval(unknownInputStr);
}
{% endhighlight %}

In a language runtime where parsing time counts towards total execution time,
we aspire to perform representation optimizations during parsing. This
limits the heft of the optimizations that might be undertaken: heavyweight
analyses lead to better optimizations, but do not pay for themselves.

In practice, this gives us the following tiers of representations on a
per-binding basis, sorted from most to least optimized.

Elision
: The binding is known to always hold the same value and is not
  accessed dynamically. It is treated as if it were a value.

On the stack frame
: The binding is not closed over by an inner function or accessed
  dynamically. It may be put on the stack frame and accessed via constant
  offset.

On the environment

: The binding is closed over by an inner function or accessed dynamically. It
  must be put in a heap-allocated environment record that maps binding names
  to values and persists beyond the lifetime of the frame. During execution,
  environments exist in a chain from outermost scope to innermost scope.

  Dynamic accesses are via name, and non-dynamic access are via a coordinate:
  a pair of natural numbers (_hops_, _slot_). _Hops_ is the number of
  environments to skip on the environment chain, and _slot_ is the known
  offset for the binding in the environment's map.

We ignore elisions for the rest of the post. A production parser can only
afford cheap, weak analyses to determine if a binding may be elided and thus
very rarely elides any bindings.

## Detecting If a Binding Is Closed Over During Parsing

If a binding is closed over or dynamically accessed, it must reside on the
environment. Otherwise it may reside on the stack frame. Detecting the
presence of dynamic accesses boils down to detecting the presence of
constructs like `eval` in the source, which is by and large
uninteresting. Determining whether a binding is closed over is the more
interesting problem.

Recall that the parser must be fast. Therefore determining closed-over
bindings must be cheap and ideally should occur as the parsing happens. Herein
lies some art. ﬁrst an obvious solution is presented, then a solution that I
believe to be a reasonable tradeoff between performance and simplicity.

JS adds two oddities.

The ﬁrst is that declarations are hoisted. In the following example, `x` is
closed over by `g`.

{% highlight js %}
function f() {
  function g() { x = 42; }
  var x;
}
{% endhighlight %}

This would be a non-issue if the analysis were performed as a separate pass
instead of during parsing itself. During parsing itself, there is no guarantee
uses of the binding precede declarations in source text.

The second is that JS has both block-scoped declarations in the forms of `let`
and `const` as well as function-scoped declarations in the form of `var`. This
means a function can close over a binding declared in an inner scope. In the
following example, `x` is closed over by `g`, because `var` declarations are
visible in the entire function.

{% highlight js %}
function f() {
  function g() { x = 42; }
  try { foo(); } catch (e) { var x; }
}
{% endhighlight %}

## The Obvious Algorithm

The obvious solution is to keep a stack of sets of names during parsing. The
algorithm is as follows.

* * *

*<center>Algorithm 1</center>*

> Let _Declared_, _Used_, and _Captured_ range over sets of names, _Tag_ be `'block'` or
> `'function'`, and ScopeStack be a stack of tuples of (_Tag_,&nbsp;_Declared_,&nbsp;_Used_,&nbsp;_Captured_).

<ol type="1">
<li>When the parser reaches a new function scope,&nbsp;push the tuple of empty sets (<code>'function'</code>,&nbsp;∅,&nbsp;∅,&nbsp;∅) onto ScopeStack.</li>

<li>When the parser reaches a new block scope, push the tuple of empty sets (<code>'block'</code>,&nbsp;∅,&nbsp;∅,&nbsp;∅) onto ScopeStack.</li>

<li>When a <code>var</code> declaration is parsed, add its binding name to the <em>Declared</em> set of all tuples on ScopeStack from the top tuple to the nearest tuple, inclusive, with a <code>'function'</code> <em>Tag</em>.</li>

<li>When a lexical declaration is parsed (e.g., a <code>let</code> or <code>const</code> declaration), add its binding name to the <em>Declared</em> set of the topmost tuple in ScopeStack.</li>

<li>When the parser reaches the end of a scope,

<ol type="a">
<li>Pop the topmost tuple (<em>tag</em>,&nbsp;<em>declared</em>,&nbsp;<em>used</em>,&nbsp;<em>captured</em>) from ScopeStack.</li>
<li>Assert that <em>captured</em> ⊆ <em>used</em>.</li>
<li>If ScopeStack is not empty:
<ol type="i">
<li>Let <em>free</em> be the set <em>used</em> - <em>declared</em>.</li>
<li>Add all names in <em>free</em> to the used set of the topmost tuple of ScopeStack.</li>
<li>If tag is <code>'function'</code>, add all names in <em>free</em> ∪ <em>captured</em> to the <em>Captured</em> set of the topmost tuple of ScopeStack.</li>
<li>If tag is <code>'block'</code>, add all names in <em>captured</em> to the <em>Captured</em> set of the topmost tuple of ScopeStack.</li>
<li>Mark all names in the set of <em>declared</em> ∩ <em>captured</em> as closed-over.</li>
</ol>
</li>
</ol>
</li>
</ol>

* * *

This algorithm answers the question, for each binding name encountered during
parsing, whether it is closed over. The ﬁrst concern above of declaration
hoisting is solved via delaying computation of whether a binding name is
closed over to when the parser reaches the end of a scope, viz. step 5. The
second concern above is addressed by step 4.

This algorithm is slow. Set difference, union, and intersection for every
scope is expensive. Moreover, these operations are performed on every
encountered identiﬁer, while the question of "is the binding closed over"
need only be determined for the binding names. The number of binding names is
typically much smaller than the number of identiﬁers.

## The Scope Numbering Algorithm

To reﬁne the algorithm and do better, we rely on the insight that we need not
propagate whether an identiﬁer is used in inner functions for all
identiﬁers. Instead, we only need to do so for binding names. That is, the
_used_ and _captured_ sets in algorithm 1 may be tracked more eﬃciently. A
list of identiﬁer uses may be gathered and kept up-to-date for each
identiﬁer as we parse. No further computation need be done on this list of
uses until the parser reaches the end of a scope with binding names, in which
case the question of "is the binding name closed over" may be answered by only
consulting the lists kept for the binding names.

What then is the suﬃcient information for this list of uses to determine
whether a binding name is closed over? Intuitively, per the algorithm 1 above,
it is whether the binding name occurs in the captured set at the end of the
binding name's scope. To eﬃciently represent the captured set, scopes and
functions in the source may be monotonically numbered in textual order. This
gives the following invariant:

> Let a scope the parser has not yet ﬁnished parsing be called a live
> scope. At any particular point during parsing, a live scope numbered _S_
> encloses a live scope numbered _P_ if and only if _S_ < _P_.

Mutadis mutandis, the same holds for function numbers.

More formally, a use is encoded as a pair of numbers (_F_,&nbsp;_S_) where _F_
ranges over function numbers and _S_ ranges over scope numbers. A binding name
is closed over if, when the parser reaches the end of the name's scope in a
function numbered _F<sub>c</sub>_, there is a use (_F_,&nbsp;_S_) in the name's use
list such that _F_ > _F<sub>c</sub>_. Care must be taken to remove stale uses
from the list for scopes we have ﬁnished parsing. This is done by, upon the
parser reaching the end of a scope numbered _S<sub>c</sub>_, removing all
entries (_F_,&nbsp;_S_) from the list of uses such that _S_ ≥ _S<sub>c</sub>_. The
list of uses is thus is accessed only in ﬁFO fashion, resulting in cheap
upkeep.

Putting the above together, the full algorithm is presented below.

* * *

*<center>Algorithm 2</center>*

> Let _FunctionId_ and _ScopeId_ range over natural numbers, _Declared_ range
> over sets of names, _Tag_ be `'block'` or `'function'`, DeclaredStack be a
> stack of tuples (_Tag_,&nbsp;_FunctionId_,&nbsp;_ScopeId_,&nbsp;_Declared_), UsedMap be a map
> of names to stacks, and _functionCounter_ and _scopeCounter_ be 0.

<ol type="1">

<li>When the parser starts parsing a new function scope,
<ol type="a">
<li>Let <em>functionCounter</em> be <em>functionCounter</em>+1.</li>
<li>Let <em>scopeCounter</em> be <em>scopeCounter</em>+1.</li>
<li>Push the tuple (<code>'function'</code>,&nbsp;<em>functionCounter</em>,&nbsp;<em>scopeCounter</em>,&nbsp;∅) onto DeclaredStack.</li>
</ol>
</li>

<li>When the parser starts parsing a new block scope,
<ol type="a">
<li>Let <em>scopeCounter</em> be <em>scopeCounter</em>+1.</li>
<li>Push the tuple (<code>'block'</code>,&nbsp;<em>functionCounter</em>,&nbsp;<em>scopeCounter</em>,&nbsp;∅) onto DeclaredStack.</li>
</ol>
</li>

<li>When a <code>var</code> declaration is parsed, add its binding name to the <em>Declared</em>
set of all tuples on DeclaredStack from the top tuple to the nearest tuple
with a <Code>'function'</code> <em>Tag</em>.</li>

<li>When a lexical declaration is parsed (e.g., a <code>let</code> or <code>const</code>
declaration), add its binding name to the <em>Declared</em> set of the topmost tuple
in DeclaredStack.</li>

<li>When the parser encounters an identiﬁer <em>u</em>,
<ol type="a">
<li>Let (<em>tag</em>,&nbsp;<em>functionId</em>,&nbsp;<em>scopeId</em>,&nbsp;<em>declared</em>) be the top tuple of DeclaredStack.</li>
<li>If <em>u</em> is found in UsedMap, append the pair (<em>functionId</em>,&nbsp;<em>scopeId</em>) to UsedMap[<em>u</em>].</li>
<li>Otherwise, let UsedMap[<em>u</em>] be the singleton stack [(<em>functionId</em>,&nbsp;<em>scopeId</em>)].</li>
</ol>
</li>

<li>When the parser reaches the end of a function or block scope,
<ol type="a">
<li>Pop the topmost tuple (<em>tag</em>,&nbsp;<em>functionId</em>,&nbsp;<em>scopeId</em>,&nbsp;<em>declared</em>) from DeclaredStack.</li>

<li>For each name <em>d</em> in <em>declared</em> that is found in UsedMap,
<ol type="i">
<li>Let <em>uses</em> be UsedMap[<em>d</em>].</li>
<li>Mark <em>d</em> as closed over if, for the topmost tuple (<em>innermostFunctionId</em>,&nbsp;<em>innermostScopeId</em>) of <em>uses</em>, <em>innermostFunctionId</em> &gt; <em>functionId</em>.</li>
<li>Pop all tuples from <em>uses</em> until the topmost tuple (<em>innermostFunctionId</em>,&nbsp;<em>innermostScopeId</em>) is such that <em>innermostScopeId</em> &lt; <em>scopeId</em>.</li>
</ol>
</li>
</ol>
</li>

</ol>

* * *

For the formally inclined, this algorithm is an imperative, single-pass moral
equivalent to variable renaming, with JS allowances added.

I invite you to take a look at the implementation [inside
SpiderMonkey](http://searchfox.org/mozilla-central/rev/76609a05d6ef7ba4223ed79e479c73fb2543a107/js/src/frontend/Parser.h#568-719).

## Bonus Caveat: Pooled Allocations

I should like to stress that pooled allocation, especially for sets of names
as used in DeclaredStack in algorithm 2, is important for performance. The
usual program has a low maximum number of nested scopes but has many
sibling scopes at each depth. Since the set of declared names need not be kept
after a scope has ﬁnished parsing, pooling allocations of data structures
used in LIFO fashion and reusing them during parsing saves much allocation
overhead.

## Conclusion

In summary, I have presented an algorithm for determining whether binding
names are closed over during parsing for JavaScript. That information is used
to determine binding representation. Binding names that are not closed over
may be stored on the stack frame, and those that are closed over must be stored
on the environment. Dynamic constructs like `eval` forces all
storage to be on the environment. I hope the lesson extends to runtimes of
other dynamic languages for which parsing time is part of the total execution
time.
