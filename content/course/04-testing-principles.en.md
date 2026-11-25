+++
title = "Testing in JS, Part 1: Principles"
weight = 4
+++

Code: `tests/preset.test.js` (the first ~90 lines — preset normalization plus
one reducer test), `store.js` as the object under test (already covered in
chapter 3).

## Theory

### Why automated tests are needed

While a program is small, its behavior can be checked by hand: run it, click something, look at the result. As it grows, that stops working — the number of possible state combinations grows faster than the ability to hold them all in your head, and any change in one place can quietly break behavior somewhere else that, at first glance, has nothing to do with it. An automated test is a fixed, repeatable check of one such combination: written once, it runs in milliseconds as many times as you like, and reports a regression the moment the developer introduces it, rather than whenever a user happens to stumble onto it.

### The testing pyramid

Not every automated check costs the same or covers the same thing. In the mid-2000s, Mike Cohn proposed picturing them as a pyramid: at the base, numerous, fast, narrow unit tests (a single function, no browser, no network, no filesystem); at the top, a handful of slow, broad end-to-end tests (the whole application, through a real browser); in between, integration tests (several modules together, but not yet the whole application). The shape of the pyramid says that the bulk of your checks should sit in the narrow, fast bottom layer, while the few broad tests at the top act as insurance, not as the primary tool.

In `JoinSheets` this shape shows up directly in the structure of the `tests/` directory. `tests/preset.test.js`, `tests/entities.test.js`, `tests/formula.test.js`, `tests/dataset.test.js` are the bottom, most numerous layer: pure module functions (`normalizePreset`, `reducer`, formula `parse`/`evaluate`), called directly from Node, with no reference to `document`/`window` whatsoever. The top layer is the subject of the next chapter: `tests/smoke/dom-smoke.js` checks that the application as a whole at least boots and the main path works, without getting into the details of any individual function. There's no formally separate integration layer between them in the project, but part of `preset.test.js` effectively plays that role — the test "reducer: preset/toggleSource creates a join," for instance, touches several things in `store.js` at once (normalization and a specific reducer case), not one isolated function.

### What makes a function easy to test

Functional programming formalized, a long time ago, the concept that answers directly what turns code into an easy or a hard target for an automated check — referential transparency: an expression is transparent if it can be replaced by its result without changing the meaning of the program, which is possible exactly when the function computing it always gives the same output for the same input and produces no side effects. A pure function performs no I/O — it doesn't read `Date.now()`, doesn't touch the DOM, doesn't send a request. Such a function is cheap to test: the test simply compares the input to the expected output, without worrying about any hidden state outside it.

The reducers in `store.js` are a rare example of code achieving this property not partially, but fully, as we already saw in the previous chapter: `reducer(state, action)` is deterministic, doesn't mutate its arguments, and reaches for nothing external. The contrast here is striking: `view-table.js::renderGrid` is practically impossible to unit-test as written — it needs a `document`, it reads module-level variables (`currentState`, chapter 7), and it creates real DOM nodes. It's exactly this contrast that gives rise to the need for smoke tests and the fake DOM, the subject of the next chapter.

### Comparing values: equality and structural equality

Nearly every testing system — regardless of language — distinguishes two different questions: "is this the very same object?" and "do these two values look the same in content?" In `node:assert` that's `assert.equal` (a comparison close to `==`, correct for primitives) versus `assert.deepEqual` (a recursive comparison of the structure of objects and arrays, not of references to them): `assert.deepEqual({x: 1}, {x: 1})` passes, because the shape is the same, even though they're two different objects in memory, while `assert.equal({x: 1}, {x: 1})` fails, because `==` compares exactly the reference for objects. The same pair of concepts shows up in other ecosystems under other names — `toBe`/`toEqual` in Jest, `assertEqual` in Python's `unittest` (where comparison for the built-in containers is already structural), `assert_eq!` in Rust (structural comparison, if the type implements `PartialEq`, usually via `#[derive(PartialEq)]`). `JoinSheets`'s tests use not the loose `node:assert`, but its strict variant, `node:assert/strict` (`const assert = require('node:assert/strict')` in `preset.test.js`) — it replaces `==`/`deepEqual` with `===`/`deepStrictEqual`, and the latter additionally compares the prototypes of the compared objects, not just their fields. This difference will come up not abstractly, but in a concrete bug — in the next chapter, where it separates objects coming from different VM contexts.

### Arrange / Act / Assert

Any unit test, in whatever system it's written, usually boils down to three steps: prepare the input data (Arrange), perform the action under test (Act), compare the result to what's expected (Assert). This isn't specific to JavaScript — the same three-part shape is recognizable in tests written in Python, Rust, or Java — but it's worth keeping explicitly in mind while reading any `JoinSheets` test:

```js
test('normalizePreset: description is kept as-is if given as a string', () => {
  const p = App.normalizePreset({ description: 'January client report' }); // Act (on a ready-made Arrange — the input data right in the call)
  assert.equal(p.description, 'January client report');                    // Assert
});
```

Here Arrange is trivial — the input object is written right into the call line. In more involved tests (such as the `reducer: preset/toggleSource` test, lines 85-90), Arrange turns into its own, preceding step: a `state` with a suitable `manifest`/`preset` has to be assembled before calling the action the test actually checks.

### Test doubles

Not every dependency of the code under test is convenient or necessary to call for real — an external API, a timer, a browser DOM. Martin Fowler systematized the vocabulary the industry settled on for replacements of such dependencies, under the umbrella term test double (by analogy with a stunt double in film), splitting it into four different roles that are easy to confuse with each other but which solve different problems: a **stub** — a placeholder that just returns a fixed value, with no check that it was ever even called; a **mock** — a placeholder that additionally checks it was called the way you expected (with the right arguments, the right number of times); a **fake** — a working, but simplified, implementation of the real dependency, not just "return 42," but real, if lightweight, logic; a **spy** — a wrapper around the real function that remembers how it was called, but doesn't change its behavior. The next chapter will need exactly a fake: `makeNode()` inside `dom-smoke.js` isn't jsdom (a full DOM implementation), but a minimal, genuinely working surrogate (`appendChild` really does push children into an array, `setAttribute` really does remember something), built for exactly the needs `JoinSheets`'s bootstrap requires, and nothing more.

### Further reading:

- **Node.js, Test runner** — the official `node:test` documentation, including
  automatic file discovery and the `--test` flag:
  https://nodejs.org/api/test.html
- **Node.js, `assert`** — the full set of checks; note the "Strict assertion
  mode" section, which is what `node:assert/strict` gives you:
  https://nodejs.org/api/assert.html
- **Node.js, Modules: CommonJS** — `require`/`module.exports`, the second
  "world" `JoinSheets`'s files live in (the first being the browser `<script>`):
  https://nodejs.org/api/modules.html
- Martin Fowler, **Test Double** — where the terms stub/mock/fake/spy come
  from, and how they actually differ:
  https://martinfowler.com/bliki/TestDouble.html
- Martin Fowler, **Test Pyramid** — a restatement and further development of
  Mike Cohn's idea:
  https://martinfowler.com/bliki/TestPyramid.html
- Jest, **Expect** — `toBe` (reference/primitive) versus `toEqual`
  (structural) in another, very widely used testing system:
  https://jestjs.io/docs/expect#toequalvalue
- MDN, **Pure functions** (in context — why they're easier to test); the
  general idea is covered well in the functional-programming glossary entry:
  https://developer.mozilla.org/en-US/docs/Glossary/Pure_function
- **The Rust Book, ch. 11** "Writing Automated Tests" — `#[test]`,
  `assert_eq!`, test organization:
  https://doc.rust-lang.org/book/ch11-00-testing.html

## Code

## Practice

## DevTools

## Rust

## Self-Check
