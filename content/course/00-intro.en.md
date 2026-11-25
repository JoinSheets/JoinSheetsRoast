+++
title = "Introduction"
description = "Who this course is for, the learning tracks, the study format, the codebase, and the course program."
weight = -1
+++

## Who this course is for

The course is aimed at an advanced audience: a general IT background and some programming practice (JS, Rust, or other languages), knowledge of web-development basics (HTML, CSS), and an understanding of how a browser works internally. And, of course, the desire to dig deeper: not just to read, but to gain practical skills, to understand a concept in isolation from the noise of business logic, to see it live in real code, to get hands-on with the developer tools, and, where possible, to rewrite the code your own way.

## Learning tracks

The course consists of several main tracks that run through every chapter, plus one bonus track:

1. **Theory** — in-depth programming concepts, computer-science fundamentals, and how modern browsers work, with brief historical digressions: where a given idea came from, what problem it solved, and how it was reinvented in different languages and frameworks.
2. **Code** — every concept from the theory is reinforced by walking through the code of a real project: genuine "messy and alive" working code, with all its trade-offs and departures from "best practice" wherever the specifics call for it.
3. **Practice** — short 15-20 minute exercises, in a quick-task format.
4. **DevTools** — alongside the code walkthrough, chapter by chapter, we get to grips with the browser's developer tools: from Sources and Console to Performance and debugging a WASM module.
5. **Bonus track: Rust** — additional practice and a final project: we build a module in Rust to boost performance and hook it up to the application through WASM.

You're free to work through these tracks in parallel and in full (recommended), or to pick only some of them (for example, theory and code, or Rust).

## Study format

Each chapter is meant to be worked through unhurriedly over roughly one week:

| Day | What you study |
|---|---|
| 1 | Theory: the concept, why it's needed, what problem it solves. At the end — a section with links to further reading on the chapter's topic |
| 2-3 | Code: a line-by-line walkthrough of the specific places in the project where it's used |
| 4-5 | Practice: exercises to check your understanding of the code, DevTools exercises, translating small code snippets into Rust |

The whole course is designed for 10 weeks with no fixed schedule or deadlines; clarifications, expanding the theory around specific spots, and fine-tuning the exercises happen on request, as needed, along the way.

## Codebase

The course's practical assignments use the [`JoinSheets`](https://github.com/JoinSheets/JoinSheets) project — a JS data viewer with a relatively small codebase, which lives in the repository [`https://github.com/JoinSheets/JoinSheets`](https://github.com/JoinSheets/JoinSheets)
(you can get to the project's repository from a link at the bottom of any course page).

The project's documentation is available on [GitHub Pages](https://JoinSheets.github.io/), the project's [demo](https://joinsheets.github.io/demo/) is hosted there as well, and it's also reachable from any course page — just click the logo in the top-left corner.

All file/line references in this course are pinned to:
```
commit <to do>
version <to do>
```

If the JoinSheets project repository moves forward, the links in this course may drift out of sync with the current version. The links are not updated or kept in sync; if needed, you can check against `git show
<to do>:<file>` in a clone of `JoinSheets`.

## Technical stack and tools

Besides language practice, the course is designed to teach the developer tools of modern web browsers. To cover this tooling in full, rather than just its breakpoints, every chapter introduces something new:

| Chapter | New in DevTools |
|---|---|
| 1 | Sources (file tree), Console (object inspection), Network, Overrides |
| 2 | Elements (live DOM tree, attributes vs. properties) |
| 3 | Sources: breakpoints, Scope/Watch, call stack |
| 4 | Node.js `--test` runner (not DevTools) |
| 5 | Sources: Coverage |
| 6 | Elements: DOM breakpoints (break on subtree modification) |
| 7 | Sources: Scope panel (closures), Console `getEventListeners()` |
| 8 | Elements: Event Listeners tab, capture vs. bubble |
| 9 | Performance (profile, flame chart, FPS) |
| 10 | Sources/Memory — debugging the WASM module |

The DevTools exercises are meant to be read and carried out at a workstation equipped with any modern browser that has a developer mode.

Optional parallel Rust track: a short "rewrite this code in Rust" exercise appears in almost every chapter (except chapter 1, which is an orientation chapter, and chapter 8, a browser-only topic with no comparable Rust functionality), plus the chapter-10 capstone.

Working through the Rust track requires an installed Rust compiler and an IDE for working with it.

## Course program

| # | Topic |
|---|---|
| [01](01-bootstrap-without-bundler/) | Loading an app without a bundler: `<script>` tags, the global namespace, ES modules |
| [02](02-dom-by-hand/) | The DOM by hand: `util.js::el()` — nodes, attributes vs. properties, handlers |
| [03](03-state-and-reducer/) | State and the reducer: `store.js` — actions, pure functions, `subscribe` |
| [04](04-testing-principles/) | Testing in JS, part 1: principles, `node:test`, what to test in `store.js` |
| [05](05-smoke-tests-fake-dom/) | Testing in JS, part 2: smoke tests and the fake-DOM pattern |
| [06](06-render-without-vdom/) | Rendering without a Virtual DOM: `main.js::render()` — full subtree rebuilds |
| [07](07-closures-and-singletons/) | Closures and module singletons: `view-table.js` (`currentState`/`currentDispatch`) |
| [08](08-event-delegation/) | Event delegation and working with DOM events |
| [09](09-performance-and-event-loop/) | Performance and the event loop: row-render batching, the Performance profile |
| [10](10-capstone-rust-wasm/) | Capstone: `formula.js` → Rust + `wasm-bindgen` → WASM, measuring the gain |
