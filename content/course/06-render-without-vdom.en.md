+++
title = "Rendering Without a Virtual DOM"
weight = 6
+++

Code: `main.js::render()` (lines 193-277) + `store.subscribe(render)` (279).

## Theory

### The cost of updating the DOM, and two answers to it

Operations that genuinely change the rendered page — inserting or removing a node, changing an attribute that can affect geometry — aren't free: they're followed by layout and paint, and both of these stages cost noticeably more than running the ordinary JS code that triggered them. Any nontrivial interface where state changes often is forced, one way or another, to answer the same question: how do you minimize the number of such expensive operations, when the mere fact that "something changed" is only known at the level of "some state changed," not at the level of "here are these three specific DOM nodes."

The approach that became widely known thanks to React (2013) and the `virtual-dom` library, released around the same time, is to build, on every state change, not a real DOM but a lightweight, plain-JS description of the interface tree, commonly called a virtual DOM. Such a description is an ordinary JS object, whose creation and comparison cost almost nothing next to real operations on the browser's tree. The library compares the new description with the old one and computes the minimal set of real changes — this node's text changed, that one's attribute got updated, this element needs to be inserted — and then applies only those to the real DOM. The task of "finding the minimal set of operations that turns one tree into another" is a classic algorithmic topic (tree edit distance), but its exact solution is expensive in its own right; every practical VDOM implementation applies heuristics rather than an exact algorithm — for instance, assuming that sibling nodes with a matching type and key (`key`) correspond to each other, without checking every possible permutation.

`JoinSheets` answers the same question in exactly the opposite way: no intermediate description and no diffing at all. On every change that should show up on screen, the code simply clears the relevant DOM node (`clear()`) and builds it again through `el()` (chapters 2-3). The cost of this solution is symmetric with its simplicity: diffing costs nothing at all, because there isn't any, but any change — even one touching a single row out of thousands in a table — costs the same as a full rebuild of the entire node. This is the direct reason chapter 9 (row-render batching) exists: without diffing, there's no way to "cheaply" update one specific row using the rendering mechanism itself — either every node gets updated at once, or, for a special case, a separate, narrow path has to be invented around the main mechanism, which in `JoinSheets` became `renderMore`.

### `shellChanged` — a manual, coarse version of diffing

```js
var shellChanged = state.route !== lastRoute || state.dataset !== lastDataset
  || state.advOpen !== lastAdvOpen || state.exportOpen !== lastExportOpen;
```

This isn't diffing in the VDOM sense (comparing a tree node by node), but one coarse boolean flag: "did anything change that requires rebuilding the whole page shell?" — if not, a narrower update is enough (`root.replaceChild`, see below), or nothing in the shell needs to be touched at all.

For the `dataset` field, this flag has to answer a specific question: did a new build actually happen, rather than just one that looks "similar" to the previous one? Reference comparison (`!==`) answers that directly and reliably: `build/success` in `store.js` always creates a new `dataset` object, regardless of whether its shape happens to match the previous one — so reference inequality unambiguously means "there was a new build," and equality means "nothing changed." Any weaker check, based not on the reference but on the shape of the data (say, a pair of "column count + row count"), risks a false match: two builds that are genuinely different — with a different `as_of` slice or a different preset name, but the same set of columns and the same row count — would be indistinguishable to such a check, even though the page header (which also displays the preset name and the slice) is required to show the actual current build. This is the same idea that already appeared in chapter 3 for domain dispatching via `=== state`: the invariant "any real change creates a new object" makes reference comparison simultaneously the cheapest and the most reliable way to answer "did something change?"

### Further reading:

- **React, Reconciliation** — how diffing works in the best-known VDOM
  implementation, and what heuristics it uses (including the role of `key`):
  https://legacy.reactjs.org/docs/reconciliation.html
- **React, "Render and Commit"** — a modern write-up of the two phases
  (compute, then apply to the DOM):
  https://react.dev/learn/render-and-commit
- MDN, **Node.replaceChild**:
  https://developer.mozilla.org/en-US/docs/Web/API/Node/replaceChild
- MDN, **Window.requestAnimationFrame** — why it instead of `setTimeout`
  (in detail in chapter 9):
  https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame
- MDN, **Critical rendering path** — what the browser does after your JS runs:
  https://developer.mozilla.org/en-US/docs/Web/Performance/Critical_rendering_path
- MDN, **Element.classList.toggle** (with the third `force` argument — exactly
  how `render()` calls it):
  https://developer.mozilla.org/en-US/docs/Web/API/DOMTokenList/toggle
- **Chrome DevTools, DOM change breakpoints** — the tool for exercise 6.4:
  https://developer.chrome.com/docs/devtools/javascript/breakpoints#dom
- **The Rust Book, ch. 6** "Enums and Pattern Matching" — for a toy `diff` in
  the Rust track:
  https://doc.rust-lang.org/book/ch06-00-enums.html
- **The Rust Book, ch. 15.1** "Box\<T\>" — why a recursive `enum Tree` needs
  a `Box`:
  https://doc.rust-lang.org/book/ch15-01-box.html

## Code

## Practice

## DevTools

## Rust

## Self-Check
