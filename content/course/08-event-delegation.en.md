+++
title = "Event Delegation and Working with DOM Events"
weight = 8
+++

Code: `view-table.js` — `attachDocCloseListener` (321-331, already seen in
chapter 7, here viewed from a different angle), the header icons (487-526),
`startResize` (369-396).

## Theory

### Event phases: the legacy of two incompatible models

When a click happens on some element, the browser doesn't just call handlers on that element — the event travels through the entire DOM tree, from the document root down to the target element and back. It wasn't always this way, and it wasn't the same everywhere: in the late 1990s Netscape and Internet Explorer implemented two mutually incompatible views of this mechanism — Netscape let an event be intercepted on its way "top-down," toward the target (capturing); Internet Explorer, only once the event had already reached the target and begun climbing back "bottom-up" (bubbling). The DOM Level 2 Events specification (2000) didn't pick one of the two models, but combined both into three sequential phases of a single event pass: capture — from `document` down to the target element; target — directly on the element itself; bubble — from the target back up to `document`. The third argument of `addEventListener(type, fn, useCapture)` determines which of the two remaining phases this particular handler fires on: `true` — during capture, before the event reaches the target and starts bubbling; `false`, or leaving the argument out (the default) — during bubble, after the target has already handled it. The vast majority of industry code uses bubble — which is exactly why the `true` that shows up in the `JoinSheets` code deserves a closer look of its own: it's a fairly rare, deliberate, and far from accidental decision.

### Event delegation

If a page has a thousand similar elements and each has to respond to a click, there are two architecturally different paths. You can attach a thousand separate handlers — each with its own closure and its own entry in the browser's memory. Or you can attach exactly one handler to a common ancestor (say, `document`) and, inside it, look at `event.target` to figure out which of the descendants the click actually landed on — usually via `event.target.closest('.the-right-class')`. The second path is called event delegation, and its savings go beyond memory: elements added to the DOM later (dynamically, after the handlers were already set up) get picked up automatically by the single handler on the ancestor — they need no `addEventListener` of their own: what fires isn't their own listener, but the ancestor's listener, in whose subtree they happen to land, regardless of when they got there. The technique is so widely accepted that React, starting with version 17, uses it not as an optional optimization but as part of its internal design: every event, regardless of the component it's described in through JSX props like `onClick`, is actually handled by one single listener on the application's root container, not on each element individually.

### Delegation and direct handlers in `JoinSheets`: one recognized criterion

`attachDocCloseListener` is the only place in the project using delegation, and it turns out to sit exactly where the technique is actually needed: a click that should close an open quick panel can land anywhere on the page, and the set of such places isn't known in advance and changes on every re-render — setting up a separate listener for every possible element would be both unrealistic and pointless. The three icons in each column's header (search, filter, sort, lines 492-508), by contrast, get ordinary, direct `onclick` handlers through `el()` — because the set of such buttons is finite and small (the number of columns times three), and — the factor that turned out to be decisive here — each handler needs to close directly over the specific column name `c` from the current `cols.map(...)` iteration. A delegated handler on the parent would have to recover that `c` by a roundabout route — say, through a `data-column` attribute on every button — which turns out to be a less direct solution with no offsetting benefit, since the scale problem itself (thousands of elements, not a few dozen) simply doesn't arise here. The criterion for when to switch, then, isn't stylistic but quantitative: delegation pays off where the set of targets is large or unknown in advance, and doesn't where it's small and every element needs a closed-over context of its own anyway.

### `closest()` — the tool of delegation

```js
if (t && t.closest && (t.closest('.quick-panel') || t.closest('.col-icon'))) return;
```

`Element.prototype.closest(selector)` walks up from `t` through its ancestors (including `t` itself) and returns the first node that matches the CSS selector, or `null`. This is the standard way for a delegated handler to tell that a click happened somewhere inside an element with a given class — without `closest()` you'd have to write your own manual `while (t.parentNode) {...}` loop.

### Further reading:

- **React, changes in version 17** — delegating every event to the root
  container as part of the internal design:
  https://legacy.reactjs.org/blog/2020/08/10/react-v17-rc.html#changes-to-event-delegation
- MDN, **EventTarget.addEventListener** — the third argument (`capture`) and
  the options object (`once`, `passive`, `signal`):
  https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener
- MDN, **Event.eventPhase** — the three phases as constants, useful for
  exercise 8.6:
  https://developer.mozilla.org/en-US/docs/Web/API/Event/eventPhase
- MDN, **Introduction to events**, the "Bubbling and capturing" section — the
  most accessible explanation of the phases:
  https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Building_blocks/Events
- MDN, **Event.stopPropagation** and **Event.preventDefault** — different
  things, often confused (`startResize` calls both):
  https://developer.mozilla.org/en-US/docs/Web/API/Event/stopPropagation
  https://developer.mozilla.org/en-US/docs/Web/API/Event/preventDefault
- MDN, **Event.target** versus **Event.currentTarget** — the key distinction
  specifically for delegation:
  https://developer.mozilla.org/en-US/docs/Web/API/Event/currentTarget
- MDN, **Element.closest**:
  https://developer.mozilla.org/en-US/docs/Web/API/Element/closest
- **Chrome DevTools, the Event Listeners tab**:
  https://developer.chrome.com/docs/devtools/dom#event_listeners
- On `passive: true` for `scroll`/`touch` listeners — useful in chapter 9,
  when we get to the scroll handler:
  https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener#passive

There are no Rust links in this chapter — the topic is entirely browser-specific (see the Rust section below).

## Code

## Practice

## DevTools

## Rust

## Self-Check
