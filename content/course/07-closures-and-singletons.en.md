+++
title = "Closures and Module Singletons"
weight = 7
+++

Code: `view-table.js`, lines 289-331 (module-level variables +
`attachDocCloseListener`) and lines 337-338, 685-689 (where they're used and
updated).

## Theory

### Closures: definition and a minimal example

Chapter 1 already introduced the general definition: a closure is a function together with the scope it captured — the one it was created in (shown there via IIFEs and encapsulation in `JoinSheets`). It's worth pinning that definition down here with its own, completely isolated example, before moving on to the subtler effect needed for the code walkthrough below:

```js
function makeCounter() {
  var count = 0;              // the outer function's variable
  return function () {        // inner function — "remembers" count
    count += 1;
    return count;
  };
}
var c1 = makeCounter();
c1(); // 1
c1(); // 2 — count didn't reset between calls, even though makeCounter() returned long ago
var c2 = makeCounter();
c2(); // 1 — c2 has its own, separate count
```

An important detail: `c1` and `c2` are different closures, each with its own copy of `count`, because every call to `makeCounter()` creates a new scope. The code in `view-table.js` works differently: its entire IIFE runs exactly once, when the file loads (see chapter 1), and every function defined inside it shares the same single copy of the variables declared at the top level of that IIFE — closer to what it would look like if `makeCounter()` were called only once for the whole application.

### The module singleton

The trick that falls out of this single IIFE call got its own name in the JavaScript community well before the language got real modules — the module pattern, which Douglas Crockford described in the early 2000s and which then found its way into every catalog of JS patterns of that era:

```js
(function () {
  var quickDraft = {};   // "module variable" — visible to every function below, invisible outside
  var renderGrid = function (state, d) { ... quickDraft ... };
  ...
})();
```

From outside the file, there's no access to `quickDraft` at all — the IIFE doesn't export it. This is what's called a module singleton: exactly one instance of state for the whole file, shared by every function inside it, unreachable from outside. In a language with a real module system (Rust, Python, modern ESM), the same idea is expressed through an explicit, built-in language feature — the `pub` keyword or its absence, the `_`-prefix convention, the module scope itself; in `view-table.js` it's achieved purely through the closure around the shared IIFE, because, as shown in chapter 1, there's no other encapsulation tool available here at all.

### A closure holds a variable, not a value

The most common mistake when first encountering closures is to think that a closure "snapshots a copy" of a value at the moment the function is created. In fact it holds a reference to the variable itself, to the same memory cell, and if that variable changes after the function is created, the closure will see exactly the new value, not whatever was fixed at some earlier point:

```js
var x = 1;
var readX = function () { return x; };
x = 2;
readX();   // 2, not 1 — readX is closed over the variable x, not over the value 1
```

This effect isn't a side nuance — it's exactly the mechanism the `currentState`/`currentDispatch` pair, examined in the code below, is built on.

### Further reading:

- MDN, **Closures** — the main text on the topic, including a section on the
  "module pattern" (exactly what's covered here):
  https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Closures
- MDN, **Scope** (glossary) and **Lexical scoping**:
  https://developer.mozilla.org/en-US/docs/Glossary/Scope
- MDN, **Memory management** — why captured variables aren't garbage-collected
  while the function that closed over them is still alive:
  https://developer.mozilla.org/en-US/docs/Web/JavaScript/Memory_management
- MDN, **Element.closest** — used in the listener examined below:
  https://developer.mozilla.org/en-US/docs/Web/API/Element/closest
- **Chrome DevTools, `getEventListeners()`** — the utility for exercise 7.6
  (works only in the DevTools console, not in page code):
  https://developer.chrome.com/docs/devtools/console/utilities#geteventlisteners-function
- "Stale closure" — the classic problem of a closure that has gone out of
  date; well described in the context of React hooks, but the underlying issue
  belongs to the language, not to the framework:
  https://react.dev/reference/react/useEffect#my-effect-uses-stale-props-or-state
- **The Rust Book, ch. 13.1** "Closures" — `Fn`/`FnMut`/`FnOnce` and `move`:
  https://doc.rust-lang.org/book/ch13-01-closures.html
- **The Rust Book, ch. 15.4-15.5** — `Rc<T>` (shared ownership) and
  `RefCell<T>` (interior mutability), both needed in the Rust track below:
  https://doc.rust-lang.org/book/ch15-04-rc.html
  https://doc.rust-lang.org/book/ch15-05-interior-mutability.html
- Python, **`nonlocal`** — the direct analog of "write to the outer closure
  variable," mentioned in chapter 1:
  https://docs.python.org/3/reference/simple_stmts.html#the-nonlocal-statement

## Code

## Practice

## DevTools

## Rust

## Self-Check
