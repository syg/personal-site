---
layout: post
title: Debugging in the Time of JITs
---

{{ page.title }}
================

Debugging, on the surface, is a feature that belies its implementation
complexity. One of its greatest challenges is debugging optimized
code. Optimizations exploit observational equivalence of program semantics to
remove and reorder code. This is anathema to debugging, which is wholly
concerned with making all state observable.

Since the JavaScript JIT wars, all major browser vendors (Apple, Google,
Microsoft, and Mozilla) ship optimizing JIT compilers in their JavaScript
engines for performance. How, then, do we effectively debug JavaScript?

Firefox's approach has been to disable the JIT when a JS script is being
debugged. This makes for poor user experience. The programmer must _a priori_
decide that the current session will be a debugging session. But suppose an
errant scenario arises: the web app encounters a corner case which results in
an infinite loop. In the current world, the programmer has no recourse but try
to reproduce the corner case (possibly intermittent!) with debugging mode
enabled. But enabling debugging means disabling the JIT, and with it reduced
performance throughout the entire execution, perhaps exacerbating the
programmer's attempt to reproduce the corner case.

We can do better. The dream of the 90s, embodied in the Self language, is
alive in JavaScript. We can adapt Self's [dynamic deoptimization via on-stack
replacement](http://bibliography.selflanguage.org/_static/dynamic-deoptimization.pdf)
technique to debug even optimized code. The core idea of a "debug mode OSR" is
simple: when debugging is required, we deoptimize and recompile JITed code on
the stack, patching return addresses as we go. One might say it is almost
elegant, if not for the exacting violence it inflicts on stack frames.

The rest of this post is a recounting of the implementation of [bug
716647](https://bugzilla.mozilla.org/show_bug.cgi?id=716647) in SpiderMonkey.

## Debugger Hooks

SpiderMonkey implements a well-designed [Debugger
API](https://developer.mozilla.org/en-US/docs/Tools/Debugger-API) that allows
programmer to implement JS debuggers in JS. It does so by reflecting engine
structures (e.g., frames, scripts, scopes) and making available hooks on those
structures (e.g., `onStep` and `onPop` on a frame) to which the programmer can
assign a plain JS function. To ensure that these callbacks are invoked, the
engine checks if there are hooks to be called at various points during
execution.

For example, suppose we set an `onStep` handler on the frame of call to
`isEven`, the source of which is shown below. Ignore the travesty of `TWO`, it
is a contrived example to demonstrate deoptimization below.

{% highlight js %}
var TWO = 2;
function mod2(x)   { return x % TWO; }
function isEven(x) { return !mod2(x); }
{% endhighlight %}

The contract of `onStep` is that it be fired after every "step" the engine
takes. In SpiderMonkey, all scripts are compiled to bytecode before execution,
and "step" means after every bytecode. In the `isEven` example, in order to
honor an `onStep` handler, the engine must check and call it after every
bytecode, listed below.

<pre class="sparse">
loc     op                                     loc     op
-----   --                                     -----   --
main:                                          main:
00000:  getgname "mod2"                        00000:  getgname "mod2"; call onStep
00005:  undefined                              00005:  undefined;       call onStep
00006:  getarg 0         set `onStep` handler  00006:  getarg 0;        call onStep
00009:  call 1           ===================>  00009:  call 1;          call onStep
00012:  not                                    00012:  not;             call onStep
00013:  return                                 00013:  return;          call onStep
00014:  retrval                                00014:  retrval
</pre>

The constraint is performance. SpiderMonkey is a three-tiered engine: an
interpreter, a baseline JIT (hereafter Baseline), and an optimizing JIT
(hereafter Ion, after its internal name, IonMonkey). To insert such a check
after every execution step across all tiers is very costly. In the
interpreter, these checks are performed unconditionally, as its performance
matters the least. In Baseline, these checks are compiled in only when
explicitly requested. Ion, which does not support compiling in debug mode
checks, is disabled by debug mode being on.

Historically, to request scripts to be compiled in debug mode with the extra
checks is a decision that must be made up front. We cannot turn on this mode
with live JS scripts on the stack, since `Debugger` hooks will not be honored
inside those live scripts which were not compiled in debug mode.

To relax this unfriendly restriction and allow debug mode to be turned on even
with on-stack scripts, we must handle both Ion and Baseline; Ion by way of
on-stack deoptimization, Baseline by way of on-stack recompilation. These
ideas are not novel; the only merit I should like to claim is their onerous
implementation.

## Deoptimization and Rematerialization

Ion, being the optimizing compiler, does not support compiling in debug mode
checks. Indeed, implementing such support is of dubious utility, as its
optimizations would make `onStep` invocations at best infelicitous. To support
debug mode, optimized code on the stack are deoptimized and bailed out to
Baseline. That is, the Ion frame on the stack is overwritten with a
reconstructed Baseline frame corresponding to the same location in which Ion
code is currently executing.

It is the author's good fortune that on-stack deoptimization is already a
critical part of Ion, or, really, any optimizing JIT for that matter. The
mercurial nature of the assumptions that the JIT depends on for optimizations
require that optimized code be able to gracefully fail over to Baseline or
Interpreter code at any fallible code point.

Let us make this concrete. Given `isEven` and `mod2` above, suppose that Ion
has compiled them with the assumption that the global variable `TWO` is an
integer. Suppose I execute the following abomination on the toplevel.

{% highlight js %}
TWO = { toNumber: function() { return 2; } };
isEven(4);
{% endhighlight %}

This causes the Ion code for `mod2` to bail out at executing `%`, which is
specialized to bare integers. The call stack is rendered below, assuming an
architecture where the stack grows down with respect to memory addresses; the
youngest frame starts at `0xf00`. Let `===` denote frame boundaries and
`---` denote values boundaries.

<pre class="dense">
|              ...              |
+============\==================+ 0xf00
| Arg0       /  4               |
+------------\------------------+
| ArgCount   /  1               |
+------------\------------------+
| This       /  undefined       |
+------------\------------------+
| Callee     /  isEven          |
+------------\------------------+
| Descr      /  Ion             |
+------------\------------------+
| ReturnAddr /  into toplevel:2 |
+============\==================+
</pre>

Ion's calling convention passes all arguments on the stack, as well as the
argument count and the `this` value for the JS function. A frame descriptor is
pushed to distinguish the kind of frame, such as Baseline or Ion (in practice
there are many other kind of frames, like stub frames for inline caches and
exit frames for re-entering the C++ runtime). `ReturnAddr` is pushed by the
`call` assembly instruction.

Doubtless, virtual machinists will point out that the frame for calling into
`mod2` is missing, if we are indeed bailing out of `mod2`. Ion, again being an
optimizing compiler, has deemed `mod2` small enough to be inlined into the
body of `isEven`. There is no actual frame for the inlined call. But Baseline
does not inline calls, and requires every frame to be visible. It is a happy
coincidence that the `Debugger` API requires the same. To bail out to
Baseline, then, is to reconstruct those inlined frames.

Ion stores enough bookkeeping information, called snapshots, to reconstruct
inlined frames as needed. Existing machinery already exists for building
Baseline stack frames from snapshots and overwriting Ion frames with those
selfsame Baseline frames.  For the current example, we replace the above Ion
`isEven` frame, which contains an inlined call to `mod2`, with the two
Baseline frames below, one for `isEven` and one for `mod2`.  The reader should
bear in mind this is an abridged and simplified picture (slots for stack
values are elided). The reality of things is more complicated, with inline
caches and stub frames; they are omitted for now and will be expanded in the
next section.

<pre class="dense">
|                       ...                       |         |
+==============\==================================+ 0xf00   |
| PrevFramePtr /                                  |---------/
+--------------\----------------------------------+
| Frame        /                                  |<--------\
+--------------\----------------------------------+         |
| Arg0         /  4                               |         |
+--------------\----------------------------------+         |
| ArgCount     /  1                               |         |
+--------------\----------------------------------+         |
| This         /  undefined                       |         |
+--------------\----------------------------------+         |
| Callee       /  isEven                          |         |
+--------------\----------------------------------+         |
| Descr        /  Baseline                        |         |
+--------------\----------------------------------+         |
| ReturnAddr   /  into toplevel:2                 |         |
+==============\==================================+         |
| PrevFramePtr /                                  |---------/
+--------------\----------------------------------+
| Frame        /                                  |
+--------------\----------------------------------+
| Arg0         /  4                               |
+--------------\----------------------------------+
| ArgCount     /  1                               |
+--------------\----------------------------------+
| This         /  undefined                       |
+--------------\----------------------------------+
| Callee       /  mod2                            |
+--------------\----------------------------------+
| Descr        /  Baseline                        |
+--------------\----------------------------------+
| ReturnAddr   /  into Baseline code for isEven:1 |
+==============\==================================+
|                       ...                       |
</pre>

After reconstructing the Baseline frames, we then jump directly into the
appropriate place in Baseline code for `mod2`, resuming execution in
Baseline. The only thing that a debug mode toggle needs to ensure, then, is
that the Baseline code for all deoptimized scripts are recompiled for debug
mode.

The astute reader might ask the following two question. One, what if there is
insufficient stack space to rewrite an existing Ion frame with Baseline
frames? Two, what of existing Baseline frames that referred to the old,
non-debug mode Baseline code? The former is addressed below, and the latter in
the next section.

For precisely the reason that there may be insufficient stack space to rewrite
an Ion frame into one or more Baseline frames, bailing out from Ion into
Baseline is only possible for the youngest frame on the stack. That is,
deoptimization is lazy. For executing scripts, this laziness is
unobservable. For the `Debugger` API, which attempts to reflect all frames, we
need a way to reflect Ion and its possibly inlined frames with no physical
representation on the stack, _even though_ by the time execution resumes in
those frames, they will have been converted to Baseline frames.

Ion and its inlined frames may have its frame slots (locals and arguments) in
registers instead of on the stack. Luckily, we can use snapshots to recover
the frame slots, as they already encode the necessary information to
reconstruct Baseline frames. The process of copying machine state with the aid
of snapshots into a heap-allocated structure for the purpose of making
optimized frames visible to the `Debugger` API is called _rematerialization_.

Rematerialized frames may be mutated by the `Debugger` API. To ensure that any
mutations are reflected in execution, rematerialized frames act as a
write-back cache for frame slots, which are written back upon bailout to the
reconstructed Baseline frame.

## Recompilation

Deoptimization alone does not enable debug mode OSR. Scripts that are only
compiled in Baseline, or Ion-compiled scripts that still have Baseline frames
on the stack are unhandled by deoptimization. For those cases, we need to
recompile the on-stack scripts and patch those stack frames, which hereafter I
shall call _on-stack recompilation_.

Baseline, being a non-optimizing compiler, admits a simple isomorphism between
the stack structures of a script compiled with debug mode on and with debug
mode off. For the purposes of this post, we can simplify further and say
that the stack structures are the exact same between the two compilation
modes. In this sense, on-stack recompilation is a degenerate case of
deoptimization. Deoptimization applies the Ion to Baseline transform on the
stack frames and jumps to recompiled Baseline code. Recompilation applies the
identity transform on the stack frames and jumps to recompiled Baseline
code. Put another way, both deoptimization and recompilation are control
operations that edit their own continuations.

It is the author's poor fortune that there does not exist pre-existing
machinery to perform on-stack recompilation. Before the implementation can be
explained in detail, let us expand the picture of the two Baseline frames
given in the previous section to be closer to reality.

The Baseline JIT compiles inline caches for most bytecode operations (for the
unfamiliar, mraleph's [blog
post](http://mrale.ph/blog/2012/06/03/explaining-js-vms-in-js-inline-caches.html)
is an excellent tutorial on the topic). For calls to other JS functions,
Baseline installs call ICs. These ICs initially start with a fallback path,
and add new inline paths as it observes them. In Baseline parlance, these
different paths are called "stubs" and have their own frame on the stack.

Continuing the existing example, the expanded Baseline frames for a call into
`isEven(4)` are rendered below.

<pre class="dense">
|                       ...                       |       |
+==============\==================================+ 0xf00 |        +=========================+
| StubPtr      /                                  |       |        |  isEven Baseline Code   |
+--------------\----------------------------------+       |        +-------------------------+
| FramePtr     /                                  |-------/        | DebugModeFlag: OFF      |
+--------------\----------------------------------+       |        +-------------------------+
| Arg0         /  4                               |       |        | main:                   | 0xa42
+--------------\----------------------------------+       |        |   ...                   |
| ArgCount     /  1                               |       |        |   enter callic 'mod2'   |
+--------------\----------------------------------+   /---|------->|   ...                   | 0xa4c
| This         /  undefined                       |   |   |        +-------------------------+
+--------------\----------------------------------+   |   |        |           ...           |
| Callee       /  isEven                          |   |   |        +-------------------------+
+--------------\----------------------------------+   |   |        | 009/0xa4c: CallIC Entry | 0xa6a
| Descr        /  BaselineStub                    |   |   |   /--->|      └ Stub             |--\
+--------------\----------------------------------+   |   |   |    +-------------------------+  |
| ReturnAddr   /  into CallIC ~> toplevel:2       |   |   |   |    |           ...           |  |
+==============\==================================+   |   |   |    +=========================+  |
| PrevFramePtr /                                  |-------/   | /-------------------------------/
+--------------\----------------------------------+   |       | |
| Frame        /                                  |<------\   | |  +=========================+
+--------------\----------------------------------+   |   |   | |  |     Shared Stub Code    |
| Fixed0       /                                  |   |   |   | |  +-------------------------+
+--------------\----------------------------------+   |   |   | \->| CallIC:                 |
|                       ...                       |   |   |   |    |   push ...              |
+--------------\----------------------------------+   |   |   |    |   call %callee_reg      |
| Stack0       /                                  |   |   |   | /->|   ...                   |
+--------------\----------------------------------+   |   |   | |  +-------------------------+
|                       ...                       |   |   |   | |  |           ...           |
+--------------\----------------------------------+   |   |   | |  +=========================+
| Descr        /  Baseline                        |   |   |   | |
+--------------\----------------------------------+   |   |   | |
| ReturnAddr   /  into Baseline code for isEven:1 |---/   |   | |
+==============\==================================+       |   | |
| StubPtr      /                                  |-------|---/ |
+--------------\----------------------------------+       |     |
| FramePtr     /                                  |-------/     |
+--------------\----------------------------------+       |     |
| Arg0         /  4                               |       |     |
+--------------\----------------------------------+       |     |
| ArgCount     /  1                               |       |     |
+--------------\----------------------------------+       |     |
| This         /  undefined                       |       |     |
+--------------\----------------------------------+       |     |
| Callee       /  mod2                            |       |     |
+--------------\----------------------------------+       |     |
| Descr        /  BaselineStub                    |       |     |
+--------------\----------------------------------+       |     |
| ReturnAddr   /  into CallIC stub code           |-------|-----/
+==============\==================================+       |
| PrevFramePtr /                                  |-------/
+--------------\----------------------------------+
| Frame        /                                  |
+--------------\----------------------------------+
| Fixed0       /                                  |
+--------------\----------------------------------+
|                       ...                       |
+--------------\----------------------------------+
| Stack0       /                                  |
+--------------\----------------------------------+
|                       ...                       |
+--------------\----------------------------------+
| Descr        /  Baseline                        |
+--------------\----------------------------------+
|                       ...                       |
</pre>

The non-abridged stack structure is a great deal more complex. The abridged
frames from the previous section are exploded into two separate frames: a
Baseline frame that stores fixed and stack slots (local variables and
temporaries), and a Baseline stub frame that does the actual call. The stub
frame is the IC frame: to call `mod2(x)`, we must first call into a `CallIC`
stub, which sets up the frame before finally calling `mod2`. When we call into
the `CallIC`, the return address pushed onto the Baseline frame is into the
main Baseline code. When we call into `mod2`, the return address pushed onto
the stub frame is into the `CallIC` stub code.

Note that the stub frame keeps a pointer to a stub data structure dangling off
the end of Baseline code. This data structure, in addition to keeping a
pointer into the corresponding stub code, contains metadata needed for IC
operation. As an IC can have multiple stubs, stubs are kept in a collection on
a corresponding IC entry. IC entries are addressable by both the machine code
address (note `0xa4c` offset in the `CallIC` entry above) and by bytecode
offset (note the `009`).

What manner of violence, then, does on-stack recompilation inflict on the
above frames and pointers? Recall that there are two steps: recompiling the
actual Baseline code and patching the frames. Recompiling is straightforward;
we simply compile another copy with debug mode on and free the original. But
now, many of the pointers on the stack are dangling, because the original
Baseline code has been discarded. The recompiled version is displayed below,
denoted by `DebugModeFlag` now being flipped on, and its code being at a different
address.

<pre class="dense">
|                       ...                       |       |
+==============\==================================+ 0xf00 |        +=========================+
| StubPtr      /                                  |       |        |  isEven Baseline Code   |
+--------------\----------------------------------+       |        +-------------------------+
| FramePtr     /                                  |-------/        | DebugModeFlag: ON       |
+--------------\----------------------------------+       |        +-------------------------+
| Arg0         /  4                               |       |        | main:                   | 0xe42
+--------------\----------------------------------+       |        |   ...                   |
| ArgCount     /  1                               |       |        |   enter callic 'mod2'   |
+--------------\----------------------------------+       |        |   call onStep           | 0xe56
| This         /  undefined                       |       |        |   ...                   |
+--------------\----------------------------------+       |        +-------------------------+
| Callee       /  isEven                          |       |        |           ...           |
+--------------\----------------------------------+       |        +-------------------------+
| Descr        /  BaselineStub                    |       |        | 009/0xe56: CallIC Entry | 0xe92
+--------------\----------------------------------+       |        |                         |
| ReturnAddr   /  into CallIC ~> toplevel:2       |       |        +-------------------------+
+==============\==================================+       |        |           ...           |
| PrevFramePtr /                                  |-------/        +=========================+
+--------------\----------------------------------+
| Frame        /                                  |<------\
+--------------\----------------------------------+       |        +=========================+
| Fixed0       /                                  |       |        |     Shared Stub Code    |
+--------------\----------------------------------+       |        +-------------------------+
|                       ...                       |       |        | CallIC:                 |
+--------------\----------------------------------+       |        |   push ...              |
| Stack0       /                                  |       |        |   call %callee_reg      |
+--------------\----------------------------------+       |     /->|   ...                   |
|                       ...                       |       |     |  +-------------------------+
+--------------\----------------------------------+   x   |     |  |           ...           |
| Descr        /  Baseline                        |   |   |     |  +=========================+
+--------------\----------------------------------+   |   |     |
| ReturnAddr   /  into Baseline code for isEven:1 |---/   |     |
+==============\==================================+       |     |
| StubPtr      /                                  |---x   |     |
+--------------\----------------------------------+       |     |
| FramePtr     /                                  |-------/     |
+--------------\----------------------------------+       |     |
| Arg0         /  4                               |       |     |
+--------------\----------------------------------+       |     |
| ArgCount     /  1                               |       |     |
+--------------\----------------------------------+       |     |
| This         /  undefined                       |       |     |
+--------------\----------------------------------+       |     |
| Callee       /  mod2                            |       |     |
+--------------\----------------------------------+       |     |
| Descr        /  BaselineStub                    |       |     |
+--------------\----------------------------------+       |     |
| ReturnAddr   /  into CallIC stub code           |-------|-----/
+==============\==================================+       |
| PrevFramePtr /                                  |-------/
+--------------\----------------------------------+
| Frame        /                                  |
+--------------\----------------------------------+
| Fixed0       /                                  |
+--------------\----------------------------------+
|                       ...                       |
+--------------\----------------------------------+
| Stack0       /                                  |
+--------------\----------------------------------+
|                       ...                       |
+--------------\----------------------------------+
| Descr        /  Baseline                        |
+--------------\----------------------------------+
|                       ...                       |
</pre>

To finish up, we must patch the dangling pointers, which, in the picture
above, are `StubPtr` for the stub frame and `ReturnAddr` for the Baseline
frame.

In another especially happy coincidence, the machine code for the various IC
stubs are shared across scripts, and thus the `ReturnAddr` in the stub pointer
is still valid and needs no patching. To wit, since the stack structure
expected differ from IC to IC (i.e., all having distinct continuations),
needing to patch this would have increased implementation burden
significantly.

To patch the dangling `ReturnAddr`, we must find the machine code location of
the point immediately after the jump into the `CallIC` in the recompiled
script. Since IC entries are indexed by both machine code and bytecode
location, the bytecode offset of the IC location is remembered from querying
the IC entry of the original Baseline script. This offset is constant across
recompile and is then used to access the IC entry in the recompiled script,
from which we then ascertain the new machine code location. In other words, by
using the constant bytecode offset as a point of reference, we can retrieve
the machine code location in the recompiled script analogous to the old
`ReturnAddr`.

To patch `StubPtr`, we must clone the old stub. Since IC stubs are added
just-in-time during execution, the recompiled Baseline code, having never been
executed, has no stubs in its IC entries. Since the machine stub code is
shared and still live, we point the cloned stub to the same code that the old
stub pointed to.

Putting this together, we arrive at a picture that is almost exactly the same
as the original, except that we now return into Baseline code that knows to
call the `Debugger` hooks.

<pre class="dense">
|                       ...                       |       |
+==============\==================================+ 0xf00 |        +========================+
| StubPtr      /                                  |       |        |  isEven Baseline Code  |
+--------------\----------------------------------+       |        +------------------------+
| FramePtr     /                                  |-------/        | DebugModeFlag: ON      |
+--------------\----------------------------------+       |        +------------------------+
| Arg0         /  4                               |       |        | main:                  | 0xe42
+--------------\----------------------------------+       |        |   ...                  |
| ArgCount     /  1                               |       |        |   enter callic 'mod2'  |
+--------------\----------------------------------+   /---|------->|   call onStep          | 0xe56
| This         /  undefined                       |   |   |        |   ...                  |
+--------------\----------------------------------+   |   |        +------------------------+
| Callee       /  isEven                          |   |   |        |           ...          |
+--------------\----------------------------------+   |   |        +------------------------+
| Descr        /  BaselineStub                    |   |   |        | 009/0xe56: CallIC Enty | 0xe92
+--------------\----------------------------------+   |   |   /--->|      └ Stub            |--\
| ReturnAddr   /  into CallIC ~> toplevel:2       |   |   |   |    +------------------------+  |
+==============\==================================+   |   |   |    |           ...          |  |
| PrevFramePtr /                                  |-------/   |    +========================+  |
+--------------\----------------------------------+   |       | /------------------------------/
| Frame        /                                  |<------\   | |
+--------------\----------------------------------+   |   |   | |  +=========================+
| Fixed0       /                                  |   |   |   | |  |     Shared Stub Code    |
+--------------\----------------------------------+   |   |   | |  +-------------------------+
|                       ...                       |   |   |   | \->| CallIC:                 |
+--------------\----------------------------------+   |   |   |    |   push ...              |
| Stack0       /                                  |   |   |   |    |   call %callee_reg      |
+--------------\----------------------------------+   |   |   | /->|   ...                   |
|                       ...                       |   |   |   | |  +-------------------------+
+--------------\----------------------------------+   |   |   | |  |           ...           |
| Descr        /  Baseline                        |   |   |   | |  +=========================+
+--------------\----------------------------------+   |   |   | |
| ReturnAddr   /  into Baseline code for isEven:1 |---/   |   | |
+==============\==================================+       |   | |
| StubPtr      /                                  |-------|---/ |
+--------------\----------------------------------+       |     |
| FramePtr     /                                  |-------/     |
+--------------\----------------------------------+       |     |
| Arg0         /  4                               |       |     |
+--------------\----------------------------------+       |     |
| ArgCount     /  1                               |       |     |
+--------------\----------------------------------+       |     |
| This         /  undefined                       |       |     |
+--------------\----------------------------------+       |     |
| Callee       /  mod2                            |       |     |
+--------------\----------------------------------+       |     |
| Descr        /  BaselineStub                    |       |     |
+--------------\----------------------------------+       |     |
| ReturnAddr   /  into CallIC stub code           |-------|-----/
+==============\==================================+       |
| PrevFramePtr /                                  |-------/
+--------------\----------------------------------+
| Frame        /                                  |
+--------------\----------------------------------+
| Fixed0       /                                  |
+--------------\----------------------------------+
|                       ...                       |
+--------------\----------------------------------+
| Stack0       /                                  |
+--------------\----------------------------------+
|                       ...                       |
+--------------\----------------------------------+
| Descr        /  Baseline                        |
+--------------\----------------------------------+
|                       ...                       |
</pre>

As a final comment, note that we only looked at one particular kind of IC,
`CallIC`. But JS can trigger function calls from many operations. Adding
things? An operand might have `toNumber` or `valueOf` hooks. Getting a
property? It might be a getter. Each "can-call" operation has its own kind of
IC. The solution presented above is agnostic to the IC kind and works for all
"can-call" ICs.

## Uses

The first use of debug mode OSR is [bug
717749](https://bugzilla.mozilla.org/show_bug.cgi?id=717749), which adds a
"Debug Script" button to the slow script dialog. Expect this feature in future
versions of Firefox.

Other useful features, which I cannot list now for my own failure of
imagination, will hopefully follow.

## Acknowledgments

I would like to thank [Jim Blandy](http://www.red-bean.com/~jimb/) for helpful
discussions and encouragement, and [Jan de Mooij](http://www.jandemooij.nl/)
for righteous code reviews.
