+++
title = "The DOM by Hand"
weight = 2
+++

Code: all of `util.js` (84 lines). Main object of study: `el()` (lines 50-69)
and `clear()` (71).

## Theory

### The document as a tree

The browser doesn't store an open page as text — it stores it as a tree of objects, the Document Object Model, or DOM. The root of the tree is the `document` object; its descendants are nodes of various kinds: elements (`<div>`, `<span>`, `<td>`, ...), text nodes (the content between tags), and a few rarer kinds (comments, processing instructions). Every node has a `parentNode`, a `childNodes` list, and a numeric `nodeType` field that distinguishes these kinds programmatically: 1 for an element (`Node.ELEMENT_NODE`), 3 for text (`Node.TEXT_NODE`), 8 for a comment (`Node.COMMENT_NODE`), 9 for the document itself (`Node.DOCUMENT_NODE`). HTML markup, which the browser parses, is one way to describe such a tree as text; but the tree exists even when not a single character of HTML is displayed — it's this tree that JavaScript works with through the DOM API.

A fragment of this tree can be built directly, call by call:

```js
var row = document.createElement('tr');
var cell = document.createElement('td');
cell.className = 'del';
cell.setAttribute('title', 'deleted (DELETE)');
cell.appendChild(document.createTextNode('∅'));
row.appendChild(cell);
```

This code works, and nothing about it is hidden — this is exactly how the DOM is built "from below." But it has a downside that becomes critical as the tree grows deeper: the structure of the result (a `row` node, with `cell` inside it, and text inside `cell`) is nowhere visible as a whole — it's smeared across a sequence of imperative assignments, and to reconstruct it in your head you have to mentally replay every line in order. Nesting in markup (`<tr><td>∅</td></tr>`) and nesting in the code that builds that markup are two different things, and with every extra level of depth, the gap between them gets more painful.

### The hyperscript pattern: a tree as a single function call

The solution several teams independently arrived at in the mid-2010s is simple: describe the tree with a single function call, where the nesting of calls matches the nesting of the result. A library with the literal name `hyperscript` (2014) set the signature shape that's been common ever since — `h(tag, props, children)` — and the name `h`, which was then adopted by `Mithril` (`m(selector, attrs, children)`), `Snabbdom` and, built on top of it, `Vue 2`'s render functions (`h(tag, data, children)`), `Preact`, and others. React arrived at the same shape not by copying, but convergently: `React.createElement(type, props, ...children)` solves exactly the same problem, and JSX, contrary to a widespread impression, is not an alternative to this approach but merely syntactic sugar over it — `<div className="x">{children}</div>` is compiled by Babel into exactly a call to `React.createElement('div', {className: 'x'}, children)` before the code ever reaches the browser.

Compare the signatures of three independent implementations of the same idea:

```js
// hyperscript (library, 2014)
h('div.warning', { title: 'caution' }, 'text')

// Mithril
m('div.warning', { title: 'caution' }, 'text')

// React.createElement — the function JSX compiles down to
React.createElement('div', { className: 'warning', title: 'caution' }, 'text')
```

The differences in detail (whether the class is part of a selector string, or a separate field of an object) don't change the essence: in each case the DOM tree is described by the nesting of calls, not by a sequence of mutations on already-existing nodes.

`util.js::el(tag, attrs, ...children)` is a member of the same family, written by hand, with no JSX and no compilation step:

```js
el('td', { class: 'del', title: 'deleted (DELETE)' }, '∅')
```

This line builds exactly the same fragment as the imperative code five lines above — but the nesting of `el()` calls now matches the structure of the result literally, one to one, which was the whole point of the pattern. Reaching for a hand-rolled implementation instead of one of the ready-made libraries above is a direct consequence of the chapter-1 constraint: with no build step there's nowhere to compile JSX, and the ready-made libraries, though available as plain `<script>` includes, would add an external dependency the project categorically doesn't accept.

### Attribute and property: two histories of the same node

A DOM node simultaneously has **attributes** — what's literally written in the source HTML text, available via `getAttribute`/`setAttribute` and always represented as a string — and **properties** of the JS object `node`, such as `node.value` or `node.checked`, which can be of any type. These two things have historically not been the same thing, and the gap between them isn't an oversight but a consequence of how the DOM evolved.

The DOM Level 1 specification (1998) described HTML as a tree staying as close as possible to the markup text: attributes were stored as name-string pairs, directly mirroring what the page's author wrote. Once pages became interactive, this representation stopped being enough: a user typing into an input field changes state that never existed, and never could have existed, in the original markup. The answer was "IDL attributes" — also known as properties of the node's JS object — a live, typed reflection of the current state, under no obligation to match whatever was once written in the text. For most pairs, the browser syncs one with the other in both directions (the `class` attribute and the `className` property) — but precisely where state is able to change independently of the markup, synchronization becomes one-directional. An input field is the clearest case: `<input value="draft">` sets only the **initial** value (technically this lands on the `defaultValue` property); as soon as the user starts typing, only `.value` changes, and the original attribute stays the same. This divergence isn't accidental, it's deliberate — it's exactly what lets a "Reset" button in an HTML form restore a field to what was originally written in the markup, rather than to whatever the user happened to type: without the split between "original value" and "current value," such a button would simply have nothing to point back to.

The same logic lies behind HTML's so-called boolean attributes (`disabled`, `checked`, `selected`, `readonly`): what matters for them is the mere presence of the attribute in the markup, not its text value — `<button disabled="false">` still sets a disabled button, because the specification reads such attributes as "present or absent," not as a boolean string. `el()` follows this rule directly: the branch `if (v === true) node.setAttribute(k, '')` sets the attribute as an empty string, not the text `"true"` — exactly the representation the browser expects for boolean attributes, and exactly what distinguishes them from any other attribute carrying a value.

`el()` deliberately makes no distinction between "set an initial value when creating the node" and "update an already-existing node": it's only ever called at creation time, through `setAttribute`, and that's enough for all of the project's needs — we'll get to the reason the second case isn't needed at all at the end of this chapter's exercise section, and in more depth in the chapter on rendering without a Virtual DOM.

### Absence of a value as a switch

Any markup generator — whatever the specific technology — runs into the same task: how to express "this attribute or this node is present only under a certain condition," without a separate branch of code for every such case. Different systems solve it differently, but the solution almost always boils down to one idea: a special, deliberately "empty" value is treated as an explicit "skip this" signal. In JSX that's `{condition && <Badge />}` — React renders nothing at all if the expression to the left of `&&` is falsy, because `false`/`null`/`undefined` as a component's child are interpreted as "there's nothing here"; Vue has a dedicated `v-if` directive for the same purpose; the `classnames` library, ubiquitous in React code, builds a CSS class string from an object, throwing out any keys whose value is falsy.

`el()` reproduces this same idea in the most economical way possible — with no dedicated syntax, directive, or library, just a check at the start of both loops inside the same function:

```js
// loop over attributes (el(), line 55)
if (v == null || v === false) return;
...
// loop over children (el(), line 64)
if (kid == null || kid === false) return;
```

The same trick, repeated verbatim in two different spots of the same function, does away with the need for a separate "if" helper for every conditional attribute or conditional child throughout the rest of the code: `el('button', { disabled: index === 0 ? false : true })` skips the attribute on `false` with no extra branching, and `el('div', {}, cond && el('span', ...))` safely "doesn't render" the `<span>` when `cond` is falsy — because `false && x` evaluates to `false` in JavaScript, exactly the value the children loop already knows how to skip. This second form — conditionally rendering a child via `&&` — is used throughout the project in exactly the form familiar from JSX code: the syntax is different (plain JS, not markup), but the problem being solved and the trick chosen to solve it are the very same.

### Further reading:

- MDN, **Document.createElement**:
  https://developer.mozilla.org/en-US/docs/Web/API/Document/createElement
- MDN, **Element.setAttribute** — and why it isn't the same as setting a property:
  https://developer.mozilla.org/en-US/docs/Web/API/Element/setAttribute
- MDN, **Boolean attributes (HTML)** — the "presence matters, not the value" rule:
  https://developer.mozilla.org/en-US/docs/Glossary/Boolean/HTML
- MDN, **HTMLElement.dataset** — `data-*` attributes as properties:
  https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/dataset
- MDN, **EventTarget.addEventListener** (we'll come back to it in chapter 8):
  https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener
- MDN, **Node.appendChild** / **Document.createTextNode**:
  https://developer.mozilla.org/en-US/docs/Web/API/Node/appendChild
- MDN, **Functions and `arguments`** — what `el()`'s variadic arity rests on:
  https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/arguments
- **React.createElement** — the same hyperscript pattern in industrial form;
  useful to compare its signature with our `el()`:
  https://react.dev/reference/react/createElement
- **hyperscript** — the library that gave the pattern its name:
  https://github.com/hyperhype/hyperscript
- **Mithril, `m()`** — the same pattern, a different project:
  https://mithril.js.org/hyperscript.html
- JSX — how it compiles down to `createElement` calls, the official write-up:
  https://react.dev/learn/writing-markup-with-jsx
- **The Rust Book, ch. 6** "Enums and Pattern Matching" — `enum` + `match` and
  exhaustiveness checking for the Rust track below:
  https://doc.rust-lang.org/book/ch06-00-enums.html

## Code

## Practice

## DevTools

## Rust

## Self-Check
