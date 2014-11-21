---
layout: post
title: Making SpiderMonkey's Debugger Just-in-Time
---

{{ page.title }}
================

Previously, I recounted the manner in which [frame surgery of on-stack
frames]({{ site.baseurl }}{% post_url 2014-05-14-debugging-in-the-time-of-jits %})
is performed in SpiderMonkey to support turning on the Debugger with live
frames. In the current post, I will describe how the [Debugger
API](https://developer.mozilla.org/en-US/docs/Tools/Debugger-API) was made
faster with that capability. The technical details of the debug mode on-stack
replacement are not prerequisite for our current exposition, which I hope is
simple and intuitive, but should you, dear reader, wish to reacquaint yourself
with the details of our methodology, please follow the link above. Most of
the implementation described below occurred in [bug
1032869](https://bugzilla.mozilla.org/show_bug.cgi?id=1032869).

## Debugger Hooks & Observability

The Debugger makes all aspects of JS execution observable by way of a set of
hooks which may be assigned with user-defined callbacks. The
execution-observing hooks and the observability requirements they admit are
described below. Execution observability is a property of *frames*, though I
abuse language below and refer to scripts as observable as well. An observable
script is a script that always pushes observable frames.

* `Debugger.onEnterFrame` is fired for every time a new JS frame is
  pushed. All scripts must be observable.

* `Debugger.onExceptionUnwind` is fired when a frame is popped during
  unwinding of an exception. Only the frames that are unwound need to be
  observable.

* `Debugger.onDebuggerStatement` is fired when a `debugger;` statement is
  encountered. Only the frame which encountered the `debugger;` statement
  needs be observable.

* `Debugger.Frame.prototype.onPop` is fired when a particular frame is
  popped. Frames are reflected as `Debugger.Frame` objects. Only those
  particular frames which have an `onPop` handler set need be observable.

* `Debugger.Frame.prototype.onStep` is fired when a particular frame is makes
  a step. Only those particular frames which have an `onStep` handler set need
  be observable.

* Setting breakpoints on a script fires breakpoint handlers. Only scripts
  which have breakpoints set need be observable.

## Just-in-Time Debugging

For the hooks enumerated above to fire, instrumentation code must be inserted
in JIT code to invoke them. Recall that SpiderMonkey is 3-tiered: interpreter,
Baseline, Ion, in order of speed of compiled code. Code running in the
interpreter and Baseline may be debugged, and code running in Ion must be made
to bail out into Baseline to be debugged. Before the current state of affairs,
entire globals are marked as being in "debug mode", which prohibited Ion
execution and caused all Baseline code compiled to be compiled with
instrumentation.

What on-stack replacement of debuggee JS frames enables is, in a happy
extension of the just-in-time philosophy, to delay the insertion of such
instrumentation until the last moment, and to only insert them into the
scripts that need them. That is, to delay the invalidation of scripts' Ion
code (discarding their Ion code as well as their inliners' Ion code) and the
recompilation of instrumentation-laden Baseline code until the last
moment. Scripts may then execute free of instrumentation in Baseline *and* Ion
until a Debugger hook needs to be fired.

For each hook above, the when and how of ensuring their observability
requirements are thus made surgical:

* `Debugger.onEnterFrame` admits an absolute, global observability over all
  scripts in the Debugger's debuggee globals. When an `onEnterFrame` hook is
  set, all optimized JIT code and baseline JIT code without instrumentation in
  those globals are discarded. On-stack Ion frames are bailed out, and
  on-stack Baseline frames are recompiled.

* `Debugger.onExceptionUnwind` requires only the frames to be unwound to be
  made observable. The technical challenge is the unwinding of an Ion frame,
  which attempts to jump directly to the corresponding `catch` or `try`
  block. Luckily, the performance degradation of unconditionally checking if
  there exists a Debugger with a live `onExceptionUnwind` hook is acceptable,
  as exceptions are, after all, exceptional cases. When there exists a live
  `onExceptionUnwind` hook, the Ion frame bails out *in situ*. Only the frames
  unwound thusly have their Ion code invalidated and their Baseline code
  recompiled.

* `Debugger.onDebuggerStatement` has a similar treatment as
  `onExceptionUnwind`. Since `debugger;` statements are rare, Baseline and Ion
  unconditionally compile instrumentation that check if there is a live
  `onDebuggerStatement` hook and if so, reflects the executing frame as a
  `Debugger.Frame`, rendering it observable (see below). Encountering
  `debugger;` statements in Ion bails out in place to Baseline.

* `Debugger.Frame` hooks and are all handled identically. Currently, having a
  `Debugger.Frame` reflection constructed ensures the referent frame is
  observable, discarding its Ion code and recompiling any on-stack Baseline
  frames. Since those hooks are all set via a `Debugger.Frame` instance, the
  underlying frame must already be observable.

* Setting breakpoints on scripts invalidates those scripts' Ion code and
  recompiles their Baseline code.

## Conclusion

Debugger is faster.
