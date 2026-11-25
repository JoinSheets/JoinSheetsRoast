+++
title = "Performance and the Event Loop"
weight = 9
+++

Code: `view-table.js` — row batching (`ROW_RENDER_BATCH`, `rowDescriptors`/
`materializeRow`, lines 533-689), `main.js`'s crash watchdog
(`requestAnimationFrame`, lines 251-275, briefly seen in chapter 6).

## Theory

### The browser's rendering pipeline

Displaying a frame on screen isn't a single operation, but a sequence of stages: Style (recomputing CSS rules for changed nodes), Layout (computing geometry — where things are and how big they are), Paint (filling in pixels), Composite (assembling layers into the final frame). JS code triggers this chain, but doesn't run it itself and doesn't run it synchronously as part of its own execution: you can change hundreds of DOM nodes in a row inside one synchronous chunk of code, and Layout/Paint for all of them happens once, later, before the next frame is drawn. At a screen refresh rate of 60 times a second, the entire chain, together with whatever JS ran before it, gets roughly 16 milliseconds (1000 divided by 60) — the so-called frame budget; going over it means dropping a frame, which the eye perceives as a jolt or a brief freeze.

### The event loop and the danger of frequent expensive handlers

The browser services events, timers, promises, and its own rendering steps through a single task-queue mechanism: macrotasks are discrete, self-contained "ticks" of execution (one event handler, one timer), between which the browser gets a chance to run a render step, if one is due; microtasks (promise handlers, in particular) run right after the current macrotask, but before the next render step. A `scroll` event, during fast mouse-wheel scrolling, generates dozens of such macrotasks per second — and if the handler for each of them runs something expensive, the browser physically doesn't have time to reach its own render step between consecutive firings: all the time gets eaten by JS, frames get dropped, and scrolling looks jerky or freezes outright.

The general, framework-independent answer to the problem of "an event fires too often for an expensive handler" is throttling (limiting the handler to one real call per fixed interval) and debouncing (delaying the call until events have stopped pouring in); both techniques have been implemented for decades by libraries like Lodash (`throttle`/`debounce`) and apply equally well to `scroll`, to `resize`, and to typing into a search field. It's exactly this problem, in its specific form — a full `store.dispatch` on every `scroll` tick, meaning reducer → `render()` → `renderGrid()` → `clear()` and rebuilding thousands of `<tr>`s (chapters 3 and 6) — that motivated the row-render batching added to the project and examined below: the problem was solved not with a generic throttle/debounce, but with a separate, narrow mechanism that bypasses the expensive `dispatch` path entirely.

### `requestAnimationFrame`: slotting in right before the next frame

`requestAnimationFrame(cb)` isn't a general-purpose timer like `setTimeout` — it's a request to call `cb` immediately before the browser starts preparing the next render frame. This gives an ordering guarantee relative to rendering that `setTimeout(cb, 0)` simply doesn't give — `setTimeout` has no connection whatsoever to what stage of the rendering pipeline the browser happens to be in when it fires. The crash watchdog in `main.js` uses this guarantee twice over — two nested `requestAnimationFrame` calls:

```js
requestAnimationFrame(function () {
  requestAnimationFrame(function () { storage.set('renderWatchdog', false); });
});
```

The first call fires before the next frame — but by that point the frame itself isn't guaranteed to have actually been painted yet: the spec guarantees "before the render," not "after" it. The nested second call waits for one more full cycle — in practice a common idiom for waiting until the browser has actually gone through the real layout/paint of the current frame, rather than merely being about to start preparing it.

### Further reading:

- MDN, **Event loop** — macrotasks, microtasks, execution order:
  https://developer.mozilla.org/en-US/docs/Web/JavaScript/Event_loop
- MDN, **Critical rendering path** — Style → Layout → Paint → Composite:
  https://developer.mozilla.org/en-US/docs/Web/Performance/Critical_rendering_path
- MDN, **Window.requestAnimationFrame** — and why it isn't `setTimeout`:
  https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame
- Lodash, **`throttle`** / **`debounce`** — the standard answers to events
  firing too often, common to any framework:
  https://lodash.com/docs/4.17.15#throttle
  https://lodash.com/docs/4.17.15#debounce
- MDN, **DocumentFragment** — why you insert in a batch rather than one node
  at a time:
  https://developer.mozilla.org/en-US/docs/Web/API/DocumentFragment
- MDN, **Element.scrollTop / scrollHeight / clientHeight** — the three
  numbers the whole load-more threshold calculation rests on:
  https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollTop
  https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollHeight
  https://developer.mozilla.org/en-US/docs/Web/API/Element/clientHeight
- **Reflow / layout thrashing** — why interleaving geometry reads with DOM
  writes is especially expensive (we managed to avoid it here, but it's worth
  knowing about):
  https://developer.mozilla.org/en-US/docs/Glossary/Reflow
- **Chrome DevTools, Performance panel** — how to read the flame chart,
  Bottom-Up, Self Time:
  https://developer.chrome.com/docs/devtools/performance
- **IntersectionObserver** — a modern alternative to a manual `scroll`
  handler for "load more once you've scrolled there"; the project doesn't use
  it, but it's useful for comparing approaches:
  https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API
- **clusterize.js** — the library whose approach to virtualization was
  studied before implementing batching in 0.31.0a1 (the project ended up
  doing something simpler):
  https://clusterize.js.org/
- **The Rust Book, ch. 8.3** "Hash Maps" — for the Rust track below:
  https://doc.rust-lang.org/book/ch08-03-hash-maps.html
- **The Rust Book, ch. 10.3** "Lifetimes" — needed as soon as `group_by`
  starts returning references into the input slice:
  https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html

## Code

## Practice

## DevTools

## Rust

## Self-Check
