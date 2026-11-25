+++
title = "Testing in JS, Part 2: Smoke Tests and the Fake-DOM Pattern"
weight = 5
+++

Code: all of `tests/smoke/dom-smoke.js` (253 lines).

## Theory

### The gap between unit tests and a real browser

The previous chapter established why pure functions are the cheapest target for an automated check, and why the reducers in `store.js` are tested without a single reference to the browser. But this cheapness has a cost: a unit test imports a function directly via `require()`, bypassing `index.html` and the order of its `<script>` tags entirely — which means it's physically incapable of catching a whole class of bugs that live precisely at the boundary between files, rather than inside any one of them. A forgotten export of a needed function into the `api` object, a scrambled file load order (chapter 1), a broken lazy reference to `window.DS_APP` (say, if `main.js` tries to read `App.viewBuild` but `view-build.js` in fact never got loaded) — none of these bugs reproduces under a direct `require()`, because in that scheme every file is always available, regardless of the order in which a browser would actually load them.

The solution to this gap, common to every ecosystem, is called end-to-end or integration testing through a real (or as close to real as possible) runtime environment — most often that means spinning up an actual browser under a tool like Selenium or Playwright and running a whole user scenario through it. `JoinSheets` takes a different path, the same one behind choosing `node:test` over `jest` in the previous chapter: not spinning up an external process or pulling in a dependency, but recreating exactly the minimum of browser behavior needed for one specific check — that the application's bootstrap, with the real file order and the real code, genuinely works from start to finish.

### `vm`: a sandbox instead of a browser

What makes this achievable without a browser is a built-in Node.js module that rarely surfaces in ordinary web-development code — `vm`. `vm.createContext(obj)` creates an isolated execution space (a "sandbox"), where `obj` becomes that sandbox's global object — the role `window` plays in a real browser. `vm.runInContext(sourceCode, ctx)` executes a piece of JS text — not a function, but literally a string of source code — inside that sandbox, as if it were one more `<script>` tag on the page. This is exactly the tool needed here: not an emulation of the DOM, but a faithful reproduction of the way a browser physically executes a sequence of `<script src="...">` tags.

### jsdom, and the deliberate choice not to use it

The most common choice for testing browser code in Node is `jsdom`, a full (if incomplete) implementation of the DOM in plain JavaScript, around which many large frameworks' test environments are built. `JoinSheets` does without it for the same reason it avoids any other dependency: `makeNode()` implements only what's genuinely needed for one pass of the bootstrap — `appendChild`/`removeChild`/`setAttribute`/`classList`/a minimal `dataset` — not the whole DOM. This is the classic choice of a fake over a full implementation (terminology from the previous chapter): cheaper to maintain, but openly and explicitly limited in what it covers. The file's own comment warns plainly: "don't treat this as a substitute for a real browser" — if functionality gets added that touches `querySelector` with real matching, or `getBoundingClientRect`, this stub won't exercise it (either it doesn't implement those methods at all, or it implements them as stubs that return nothing).

### Further reading:

- **Playwright** — a modern example of end-to-end testing through a real
  browser, for comparison with the approach `JoinSheets` deliberately didn't
  choose:
  https://playwright.dev/
- **Node.js, `vm`** — sandboxes, `createContext`/`runInContext`; note the
  warning at the start of the docs that this is NOT a security mechanism:
  https://nodejs.org/api/vm.html
- **jsdom** — what `JoinSheets` deliberately doesn't use; worth looking at the
  list of what's implemented, to get a sense of the scope of a "real" DOM:
  https://github.com/jsdom/jsdom
- MDN, **Window.sessionStorage** — what exactly the stub replaces (and how it
  differs from `localStorage`):
  https://developer.mozilla.org/en-US/docs/Web/API/Window/sessionStorage
- MDN, **Element.classList** — the interface the stub reimplements by hand:
  https://developer.mozilla.org/en-US/docs/Web/API/Element/classList
- MDN, **Object.getPrototypeOf** — why `deepStrictEqual` might not agree for
  objects from different "realms" (exercise 5.3):
  https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/getPrototypeOf
- MDN, **Monkey patching** (glossary) — the technique used to intercept
  `appendChild`:
  https://developer.mozilla.org/en-US/docs/Glossary/Monkey_patch
- **Chrome DevTools, Coverage** — the tool from exercise 5.4:
  https://developer.chrome.com/docs/devtools/coverage
- **The Rust Book, ch. 11.3** "Test Organization" — unit tests inside a module
  versus integration tests in a `tests/` directory:
  https://doc.rust-lang.org/book/ch11-03-test-organization.html

## Code

## Practice

## DevTools

## Rust

## Self-Check
