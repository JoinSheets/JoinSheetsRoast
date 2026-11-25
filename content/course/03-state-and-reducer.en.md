+++
title = "State and the Reducer"
weight = 3
+++

Code: `store.js` — `createStore` (55-66), `initialState` (176-209), the domain
reducers (224-612), and their composition in `reducer()` (614-668). No need to
go through every `case` — the structure matters more than the content of any
particular branch.

## Theory

### The problem a single source of truth solves

Any application with an interface more complex than a couple of static screens runs into the same question: where is state stored (what's selected, what's open, what's been typed) and who is allowed to change it. In applications written without much discipline, the answer is usually smeared across the whole codebase — state lives in dozens of independent variables, DOM attributes, and closures, and any event handler can change any part of it directly. This works fine while there isn't much interaction going on; but as the application grows, debugging turns into an investigation: a bug reproduces, but figuring out exactly what, and where, changed the state means setting breakpoints in every place that could theoretically have affected it.

The Flux architecture, presented by Facebook in 2014 for its own needs (at the time, for early React), offered a simple way out: make state changes possible through only one narrow channel. Every interface event becomes an **action** — a simple, self-contained description of what happened (`{type: 'string', ...payload}`); a single function — the **store** — receives such actions one after another and decides how each of them changes the one, shared application state. The same principle, independently and almost at the same time, was taken to its purest form in the Elm architecture (The Elm Architecture: Model, Update, View) — a language where a scheme like this is the only one there is, built into the language itself rather than a library on top of it. Redux, written in 2015 as an attempt to bring exactly Elm's ideas into the React ecosystem, boiled all of this down to three statements that have since become close to an industry standard: the application's state is described entirely by a single object; the only way to change it is to dispatch an action; and turning an action into a new state is the job of a pure function called a reducer — `(state, action) => newState`, which doesn't touch the state it received, but builds a new one.

`JoinSheets` implements this pattern not as a library, but directly — a few dozen lines in `store.js`, with no external dependency at all (the same reason `el()` in the previous chapter was written by hand instead of taken off the shelf). Any part of the interface that needs to learn about a state change subscribes to notifications through `subscribe` — we'll come back to this mechanism below.

### `createStore`: the observer pattern in its purest form

The mechanism of "subscribe to changes and be notified when they happen" is older than Redux itself, older than the web — it's the classic Observer pattern (also known as publish/subscribe), applied for decades in all sorts of GUI toolkits and event systems, from listeners in Java Swing to `EventEmitter` in Node.js. `createStore` implements it in an extremely compact form:

```js
var createStore = function (reducer, initial) {
  var state = initial;
  var listeners = new Set();
  return {
    getState: function () { return state; },
    dispatch: function (action) {
      state = reducer(state, action);
      listeners.forEach(function (l) { l(state, action); });
    },
    subscribe: function (l) { listeners.add(l); return function () { listeners.delete(l); }; }
  };
};
```

`state` is a variable captured by closure (chapter 7 goes into this mechanism in more depth) inside `createStore`, unreachable from outside except through `getState()`. `dispatch` is the one place in the entire application that changes this variable, and that's exactly the "one narrow channel" Flux talks about. `subscribe` returns an unsubscribe function — a closure over the specific listener `l` and the shared `listeners` set — twelve lines total, and not a single external library for a pattern that gets its own npm package in Redux.

### Immutability as a tool, not a style preference

The requirement "a reducer must not mutate the old state, it must build a new one" looks, at first glance, like a stylistic recommendation, but it has a concrete, measurable consequence: if the new state is guaranteed to be a new object on any real change, then checking "did something change?" reduces to a single reference comparison (`oldState === newState`) — with no need to recursively walk the whole state tree and compare its contents field by field. The same trick, resting on exactly the same guarantee, underlies `React.memo`/`PureComponent` — they decide whether to re-render a component by a shallow, reference-based comparison of props rather than a deep one, and that works correctly precisely when the state those props are derived from is also never mutated in place.

In `JoinSheets`, this same guarantee gets used not once, but twice, in two unrelated places: inside `reducer()` itself, for dispatching across domains (below in this chapter), and in `main.js::render()`, where the comparison `state.dataset !== lastDataset` decides whether the page shell needs to be rebuilt entirely (we'll go through this in detail in chapter 6). This isn't a coincidence, but a direct consequence of the same invariant that `store.js` maintains centrally: since every reducer is required to return a new object on any real change, any code that only reads the state can safely rely on that invariant.

### Composing reducers: an alternative to `combineReducers`

Classic Redux usually assembles one large reducer out of small ones via `combineReducers({a: reducerA, b: reducerB})` — each sub-reducer owns its own, pre-known key of the top-level state (`state.a`, `state.b`), and all of them get called on every single action.

`JoinSheets` is built differently: none of the eight domain reducers (`preset/*`, `group/*`, `sort/*`, and so on) owns a separate key — all of them read and write the same, shared, flat `state`. There are simply no keys locked to each domain, so they're called one after another, and the dispatching is resolved by the same invariant described above: if a domain returned a new object (`!== state`), it means it recognized the action it was given, and the result can be returned right away without trying the rest; if a domain returned the same `state`, that means its `default` branch fired, and the next one should be tried. This is reliable precisely because not a single `case` anywhere in the file returns `state` "as is" for a recognized action — always `Object.assign({}, state, ...)` or `setIn(state, ...)`, which guarantees a new reference.

### Further reading:

- **Flux** — Facebook's 2014 architecture that motivated this whole approach:
  https://facebookarchive.github.io/flux/docs/in-depth-overview/
- **The Elm Architecture** — an independent, extremely clean formulation of the
  same idea (Model/Update/View), which influenced Redux's design:
  https://guide.elm-lang.org/architecture/
- **Redux, Three Principles** — the canonical statements of "single source of
  truth / state is read-only / changes via pure functions":
  https://redux.js.org/understanding/thinking-in-redux/three-principles
- **Redux, `combineReducers`** — the standard way to compose reducers, which
  we compare here against "try the domains one by one":
  https://redux.js.org/api/combinereducers
- **Redux, Immutable Update Patterns** — why `Object.assign`/spread instead of
  mutation, and how to update nested structures (exactly what `util.js::setIn`
  does):
  https://redux.js.org/usage/structuring-reducers/immutable-update-patterns
- MDN, **Object.assign**:
  https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/assign
- MDN, **Equality comparisons and sameness** — the basis of the `next !== state`
  trick:
  https://developer.mozilla.org/en-US/docs/Web/JavaScript/Equality_comparisons_and_sameness
- MDN, **Set** — what `listeners` rests on inside `createStore`:
  https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set
- The **Observer** pattern (also publish/subscribe) — what `subscribe`/`dispatch`
  boils down to:
  https://refactoring.guru/design-patterns/observer
- **The Rust Book, ch. 6** "Enums and Pattern Matching" — `enum Action` +
  `match` instead of a string `switch`:
  https://doc.rust-lang.org/book/ch06-00-enums.html
- **The Rust Book, ch. 18.3** "Pattern Syntax" — for writing more elaborate
  conditional `match` arms, if you want to:
  https://doc.rust-lang.org/book/ch18-03-pattern-syntax.html

## Code

## Practice

## DevTools

## Rust

## Self-Check
