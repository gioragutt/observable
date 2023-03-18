# Observable

This document is the [explainer](https://tag.w3.org/explainers/) for the Observable interface,
a proposed addition to the WHATWG [DOM Standard](https://github.com/whatwg/dom) for more ergonomic
and composable event handling.

## Authors:

 - [Dominic Farolino](https://github.com/domfarolino)

## Table of contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Background](#background)
  - [What is an Observable?](#what-is-an-observable)
- [Problem](#problem)
  - [Userland libraries](#userland-libraries)
- [The `Observable` API](#the-observable-api)
  - [`EventTarget` integration](#eventtarget-integration)
    - [Example 1](#example-1)
    - [Example 2](#example-2)
    - [Example 3](#example-3)
    - [Example 4](#example-4)
    - [Example 5](#example-5)
  - [Further platform integration](#further-platform-integration)
- [Operators & combinators](#operators--combinators)
- [Concerns](#concerns)
- [Standards venue](#standards-venue)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Background

Observables were first proposed to the platform in [TC39](https://github.com/tc39/proposal-observable)
in May of 2015. The proposal failed to gain traction, in part due to some opposition that
the API was suitable to be a language-level primitive. In an attempt to renew the proposal
at a higher level of abstraction, a WHATWG [DOM issue](https://github.com/whatwg/dom/issues/544) was
filed in December of 2017. Despite ample [developer demand](https://foolip.github.io/spec-reactions/),
*lots* of discussion, and no strong objectors, the DOM Observables proposal sat mostly still for several
years (with some flux in the API design) due to a lack of implementer prioritization.

This repository is an attempt to again breath life into the Observable proposal with the hope
of shipping a version of it to the Web Platform.

### What is an Observable?

To cover the basics, please read [this resource](https://rxjs.dev/guide/observable). But in short,
an Observable is a glorified function: that is, it's an interface that gets constructed with a
function that can be called any number of times by calling `subscribe()` on the constructed
Observable. Subscribing to an observable synchronously invokes the function that was supplied upon
construction and sets up a new "subscription" which can receive values from the Observable
(synchronously or asynchronously) by calling a `next()` handler passed into `subscribe()`.

It is in that sense that an Observable is a glorified function. Additionally, it has extra
"safety" by telling the user exactly when:
 - It is forever finished emitting values for a particular subscription: by invoking a `done()` handler
 - It encounters an error (in which case it also stops emitting values to the subscriber) by invoking an `error()` handler
 
 To illustrate of how producers and consumers of data interact, Observables compare with other primitives
 as described by this table, which is an attempt at combing
 [two](https://github.com/kriskowal/gtor#a-general-theory-of-reactivity) different
 [tables](https://rxjs.dev/guide/observable):
 
 
 <table>
 <thead>
 	<tr>
			<th></th>
			<th colspan="2">Singular</th>
			<th colspan="2">Plural</th>
		</tr>
		<tr>
			<th></td>
			<th>Spatial</th>
			<th>Temporal</th>
			<th>Spatial</th>
			<th>Temporal</th>
		</tr>
 </thead>
	<tbody>
		<tr>
			<th>Push</th>
			<td>Value</td>
			<td>Promise</td>
			<td colspan="2">Observable</td>
		</tr>
		<tr>
			<th>Pull</th>
			<td>Function</td>
			<td>Async iterator</td>
			<td>Iterable</td>
			<td>Async iterator</td>
		</tr>
	</tbody>
</table>


<!-- A second version
<table>
 <thead>
 	<tr>
			<th></th>
			<th colspan="2">Spatial</th>
			<th colspan="2">Temporal</th>
		</tr>
		<tr>
			<th></td>
			<th>Singular</th>
			<th>Plural</th>
			<th>Singular</th>
			<th>Plural</th>
		</tr>
 </thead>
	<tbody>
		<tr>
			<td>Push</td>
			<td>Value</td>
			<td>Observable</td>
			<td>Promise</td>
			<td>Observable</td>
		</tr>
		<tr>
			<td>Pull</td>
			<td>Function</td>
			<td>Iterable</td>
			<td colspan="2">Async iterator</td>
		</tr>
	</tbody>
</table>
-->

## Problem

Observables do not introduce new capabilities — which is why they can be [polyfilled](#userland-libraries)
with full fidelity — but rather greatly enhance ergonomics around event handling, especially with the many
operators and combinators used with Observables. Consider these examples:

### Userland libraries

In [prior discussion](https://github.com/whatwg/dom/issues/544#issuecomment-1433955626),
[Ben Lesh](https://github.com/benlesh) has listed several custom userland implementations of the
Observables primitive, of which RxJS is the most popular with "47,000,000+ downloads *per week*."

 - RxJS
 - React Router
 - Redux
 - Vue
 - Svelte
 - XState
 - MobX
 - Relay
 - Recoil
 - Apollo GraphQL
 - tRPC

Given the extensive prior art in this area, there exists a public "[Observable Contract](https://reactivex.io/documentation/contract.html)" to which all userland implementations
are expected to adhere — this scenario is not unlike the [Promises/A+](https://promisesaplus.com/)
specification that was developed before `Promise`s were adopted into ES2015 as a first-class
language primitive.

## The `Observable` API

The proposed `Observable` API shape is as follows:

```cs
partial interface EventTarget {
  Observable on(DOMString type, optional AddEventListenerOptions options);
};

callback SubscribeCallback = void (Subscriber subscriber);
callback ObserverCallback = void (any value);
callback ObserverCompleteCallback = void ();

dictionary Observer {
  ObserverCallback next;
  ObserverCompleteCallback complete;
  ObserverCallback error;
  AbortSignal signal;
};

[Exposed=*]
interface Observable {
    constructor(SubscribeCallback callback);
    subscribe(Observer observer);

    // TODO: Consider operators
};
```

### `EventTarget` integration

While native Observables are theoretically useful on their own, the primary use case that we're
unlocking with them is more *ergonomic* and *composable* event handling. This necessitates
tight integration with the [`EventTarget`](https://dom.spec.whatwg.org/#interface-eventtarget) DOM
interface.

This proposal adds an `.on()` method to `EventTarget` that becomes a better `addEventListener()`.
Specifically `.on()` returns a new Observable constructed natively with a subscribe function that
essentially adds a new event listener to the target and calls the `.next()` handler with each event.

This is useful because it turns event handling, filtering, and termination, into an explicit declarative
flow that's easier to understand and
[compose](https://stackoverflow.com/questions/44112364/what-does-this-mean-in-the-observable-tc-39-proposal)
than today's imperative version, which often require nested calls to `addEventListener()` and hard-to-follow
callback chains. Consider these examples:

#### Example 1

```js
// Filtering and mapping:
element.on("click").
    filter(e => e.target.matches(".foo")).
    map(e => ({x: e.clientX, y: e.clientY })).
    subscribe(handleClickAtPoint);
```

#### Example 2

```js
// Automatic/declarative unsubscription via the takeUntil method:
element.on("mousemove").
    takeUntil(document.on("mouseup")).
    subscribe(etc => …);

// Since reduce and other terminators return promises, they also play
// well with async functions:
await element.on("mousemove").
    takeUntil(element.on("mouseup")).
    reduce((e, soFar) => …);
```

#### Example 3

```js
// Declarative:
element.on('mousemove').takeUntil(element.on('mouseup')).subscribe(console.log);

// Imperative:
const controller = new AbortController();
element.addEventListener('mousemove', e => {
  element.addEventListener('mouseup', e => controller.abort());
  console.log(e);
}, {signal});
```

#### Example 4

From https://github.com/whatwg/dom/issues/544#issuecomment-351705380:
```js
container.on('click').filter(e => e.target.closest('a')).subscribe(e => {
  // …
});
```

#### Example 5

From https://github.com/whatwg/dom/issues/544#issuecomment-351762493:

```js
// Find the maximum Y coordinate while the mouse is held down.
const maxY = await element.on("mousemove")
                          .takeUntil(element.on("mouseup"))
                          .map(e => e.clientY)
                          .reduce((y, soFar) => Math.max(y, soFar), 0);
```

We really need to mention that these are *synchronous*. https://github.com/whatwg/dom/issues/544#issuecomment-351758385
is a good example to drive the point home.

### Further platform integration

https://github.com/whatwg/dom/issues/544#issuecomment-631402455

## Operators & combinators

To prevent scope creep, this proposal certainly does not involve shipping the entire set of 
[operators included in](https://rxjs.dev/api?query=operators&type=function) popular userland
libraries like RxJS. As has been [said](https://github.com/whatwg/dom/issues/544#:~:text=What%20would%20I%20find%20to%20be%20acceptable%20criteria%20for%20a%20good%20Observable%20type%20in%20the%20platform%3F)
in [previous discussions](https://github.com/whatwg/dom/issues/544#issuecomment-351454978),
we expect userland libraries to implement and provide specific operators that integrate with the
Observable API that's central to this proposal.

With that said, an initial set of common operators that come _with_ a native Observable
API could greatly increase utility and ease adoption, so we can consider including operators that
already exist on other iterables as a part of this proposal; the exact set of operators is currently
under consideration, but we can look to TC39's [iterator helpers proposal](https://github.com/tc39/proposal-iterator-helpers)
for guidance, which adds the [following methods](https://tc39.es/proposal-iterator-helpers/#sec-iteratorprototype)
to `Iterator.prototype`:

 - `map()`
 - `filter()`
 - `take()`
 - `drop()`
 - `flatMap()`
 - `reduce()`
 - `toArray()`
 - `forEach()`
 - `some()`
 - `every()`
 - `find()`
 - maybe: `from()`[^1]

Some subset of these could be included in an initial Observables MVP, with others shipping
independently after, and more niche operators staying is userland libraries until/unless they
get the momentum to graduate and ship on the platform. In any case it is important to realize that
operators _are not_ the meat of this proposal, as thy could conceivably follow along at any time
provided there is support for the actual native Observable API, which _is_ what this proposal principally
focuses on.

## Concerns

One of the main [concerns](https://github.com/whatwg/dom/issues/544#issuecomment-351443624)
expressed in the original WHATWG DOM thread has to do with Promise-ifying APIs on Observable,
such as the proposed `first()`. The potential footgun here with microtask scheduling and event
integration. Specifically, the following innocent-looking code would not *always* work:

```js
element.on('click').first.then(e => {
  e.preventDefault();
  // Do something custom...
});
```

If `Observable#first()` returns a Promise that resolves when the first event is fired on an
`EventTarget`, then the user-supplied Promise `.then()` handler will run:
 - ✅ Synchronously after event firing, for events triggered by the user
 - ❌ Asynchronously after event firing, for all events triggered by script (i.e., `element.click()`)
    - This means `e.preventDefault()` will have happened too late and effectively been ignored

<details>
To understand why this is the case, you must understand how and when the microtask queue is flushed
(and thus how microtasks, including Promise resolution handlers, are invoked).

In WebIDL after a callback is invoked, the HTML algorithm
*[clean up after running script](https://html.spec.whatwg.org/C#clean-up-after-running-script)*
[is called](https://webidl.spec.whatwg.org/#ref-for-clean-up-after-running-script%E2%91%A0), and
this algorithm calls *[perform a microtask checkpoint](https://html.spec.whatwg.org/C#perform-a-microtask-checkpoint)*
if and only if the JavaScript stack is empty.

Concretely, that means for `element.click()` in the above example, the following steps occur:
  1. To run `element.click()`, a JavaScript execution context is first pushed onto the stack
  1. To run the internal `click` event listener callback (the one created natively by the
     `Observable#from()` implementation), *another* JavaScript execution context is pushed onto
     the stack, as WebIDL prepares to run the internal callback
  1. The internal callback runs, which immediately resolves the Promise returned by `Observable#first()`;
     now the microtask queue contains the Promise's user-supplied `then()` handler which will cancel
     the event once it runs
  1. The top-most execution context is removed from the stack, and the microtask queue **cannot be
     flushed**, because there is still JavaScript on the stack.
  1. After the internal `click` event callback is executed, the rest of the event path continues since
     event was not canceled during or immediately after the callback. The event does whatever it would
     normally do (submit the form, `alert()` the user, etc.)
  1. Finally, the JavaScript containing `element.click()` is finished, and the final execution context
     is popped from the stack and the microtask queue is flushed. The user-supplied `.then()` handler
     is run, which attempts to cancel the event too late
</details>

Two things mitigate this concern. First, there is a very simple workaround to *always* avoid the
case where your `e.preventDefault()` might run too late:

```js
element.on('click').map(e => (e.preventDefault(), e)).first()
```

...or if Observable had a `.do()` method (see https://github.com/whatwg/dom/issues/544#issuecomment-351457179):

```js
element.on('click').do(e => e.preventDefault()).first()
```

...or by [modifying](https://github.com/whatwg/dom/issues/544#issuecomment-351779661) the semantics of
`first()` to take a callback that produces a value that the returned Promise resolves to:

```js
el.on("submit").first(e => e.preventDefault()).then(doMoreStuff)
```

Second, this "quirk" already exists in today's thriving Observable ecosystem, and there are no serious
concerns or reports from that community that developers are consistently running into this. This gives
some confidence that baking this behavior into the web platform will not be dangerous.

## Standards venue

There's been much discussion about which standards venue should ultimately host an Observables
proposal. The venue is not inconsequential, as it effectively decides whether Observables becomes a
language-level primitive like `Promise`s, that ship in all JavaScript browser engines, or a web platform
primitive with *optional* consideration in other environments like Node.js (see [`AbortController`](https://nodejs.org/api/globals.html#class-abortcontroller) for example).

In previous discussion it had been decided that [WHATWG DOM Standard](https://github.com/whatwg/dom)
is the right home for Observables due to its integration with the web platform event [event system](#this) and
lack of new syntax or language capabilities. In attempt to avoid relitigating this discussion, we'd urge the
reader to see the following discussion comments:

 - https://github.com/whatwg/dom/issues/544#issuecomment-351520728
 - https://github.com/whatwg/dom/issues/544#issuecomment-351561091
 - https://github.com/whatwg/dom/issues/544#issuecomment-351582862
 - https://github.com/whatwg/dom/issues/544#issuecomment-351607779
 - https://github.com/whatwg/dom/issues/544#issuecomment-351718686

[^1]: This appears [in the TC39 proposal's `README.md`](https://github.com/tc39/proposal-iterator-helpers#fromobject) file but not the spec, so its fate is unclear.