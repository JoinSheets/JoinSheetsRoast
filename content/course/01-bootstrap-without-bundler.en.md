+++
title = "Bootstrapping an App Without a Bundler or Package Manager"
weight = 1
+++

Files for this chapter: `index.html`, `main.js` (in full), and a bit of `effects.js`.

## Theory

### Modularity and scope in JavaScript

Any language meant for programs made of several files has to solve the same problem: how will these files refer to each other, and what written in one file stays its own internal business versus becomes part of an interface the rest of the code gets to use. The solution to this problem is called modularity. In JavaScript, the history of this solution splits into "before" and "after" 2015, and before we look at how modularity is organized in the project, it's worth pinning down how it works in modern JS — without that baseline it would be unclear where the project follows the canon, where it departs from it, and why.

The starting point here is scope: the rule that determines where a variable or function name stays reachable. In JavaScript there are, historically, three such rules, and they are not equally good.

Global scope is the widest and most dangerous of the three. In the browser it corresponds to the `window` object (in Node.js, `global`): any name declared with `var` at the top level of an ordinary `<script>` tag becomes a property of that object. If `util.js` declares `var el = ...`, then once it has loaded, `window.el` exists, visible from any other file on the page. The convenience here is deceptive: two files that happen to declare the same name quietly overwrite each other — whichever loaded later wins, with no warning at all. It helps to picture this scope as a shared communal kitchen: there's one stove for everyone, and nothing stops a neighbor from moving your pot aside — or just taking it with them.

Function scope works the opposite way: anything declared inside a function simply does not exist outside that function. Before other mechanisms appeared, this was the only reliable way to hide anything in JavaScript — which is exactly where the immediately-invoked function expression (IIFE) trick comes from, covered in its own section below.

Block scope, introduced with `let` and `const` in ES2015, is already stricter: visibility is limited to a pair of curly braces. It solves a number of specific annoyances (a loop variable no longer "leaks" past the `for`), but it does not solve the "file versus file" problem: a `let` declared at the top level of an ordinary `<script>` is still shared across every script on the page — it simply, formally, doesn't become a property of `window`.

The answer to the limitations of all three rules above was ES modules (ESM) — a built-in, bundler-free mechanism for modularity, native to the language and the browser itself. A file loaded as `<script type="module" src="util.js">` gets its own scope automatically: everything declared in it is, by default, unreachable from outside, and nothing ends up on `window`. Only what's explicitly marked with the `export` keyword is handed outward:

```js
// util.js
const PRIVATE_DEFAULT = 120;              // completely unreachable from outside
export function el(tag, attrs) { ... }    // part of the module's public API
export const MIN_W = 24;
export default el;                        // the "main" export, if it's the only one
```

```js
// view-table.js
import el, { MIN_W } from './util.js';           // default export + a named one
import * as Util from './util.js';                 // the whole namespace as one object
import { el as makeEl } from './util.js';           // import with renaming
```

This mechanism has five properties that a plain set of `<script>` tags never had, and never could have.

First, load order is computed automatically: the developer doesn't list files in the right sequence — the browser itself builds a dependency graph from `import` statements and loads/executes modules in topological order. Circular dependencies don't break everything in the process; they're handled according to predefined rules.

Second, the dependency graph is static: before any module starts executing, the engine already knows what it imports and from where (the so-called linking phase, which precedes the execution phase). A typo in an import name turns into an immediate error at load time, rather than an `undefined` that quietly surfaces an hour later, on first access. The same property is what makes dead-code elimination possible: bundlers can statically see which exports of a module are never actually used, and leave them out of the final bundle without running a single line of code.

Third, every module executes exactly once, no matter how many files import it — everyone gets the same single copy. This is the module singleton; chapter 7 will show how the same effect is recreated by hand in the project, through an IIFE.

Fourth, modules always execute in strict mode (`'use strict'` is implied, no need to declare it), and `this` at the top level of a module is `undefined`, not `window`.

Finally, loading a module defers its execution, much like the `defer` attribute: parsing of the HTML document is not blocked, and the module's code only runs after the document has been built.

It's worth gathering these five points into one table — not to pit "good and new" against "bad and old," but to see which part of this functionality gets recreated by hand in the project, and which is deliberately left out:

| | Global `<script>` | ES module | `JoinSheets` |
|---|---|---|---|
| Privacy by default | no (everything on `window`) | yes | yes, via IIFE |
| Public API | whatever made it onto `window` | the `export` keyword | an entry on `window.DS_APP` |
| Load order | set by hand, in HTML | computed automatically from the graph | set by hand, in HTML |
| Missing dependency | `undefined` at runtime | error at the linking stage | `undefined` at runtime |
| Works over the `file://` protocol | yes | no (see below) | yes |

The last row of this table is exactly why the project settled on not using ES modules, but the mechanism they came to replace.

### Why `file://` closes the door on `import`/`export`

The limitation the last row refers to has nothing to do with the capabilities of the language itself — ES modules work perfectly well in any modern browser. The issue is *how* the browser loads a module: the request goes through `fetch`, and `fetch`, in turn, follows the same security rules — the same-origin policy and its partial relaxation, CORS — as any other network request made from a page.

To understand exactly what these rules forbid, we first need to pin down precisely what an origin is. An origin is a triple of scheme, host, and port — nothing more:

```
https://example.com:443/app/index.html
└─┬──┘   └────┬────┘ └┬┘
 scheme      host     port        → origin: "https://example.com:443"
```

The path (`/app/index.html`), query string, and address fragment are not part of this concept. A few consequences follow, which are worth checking against concrete pairs of addresses: `https://example.com/a.html` and `https://example.com/deep/nested/b.js` belong to the same origin, since the path is not counted; `https://example.com` and `http://example.com` are different origins, since the scheme differs; `https://example.com` and `https://api.example.com` are different too, since a subdomain is a separate host; `https://example.com` and `https://example.com:8443` differ only in port, but that alone is enough to count as different origins.

The same-origin policy is one of the fundamental security rules of the web: code loaded from one origin cannot, by default, read a response received from another. Without this rule, a tab with an arbitrary site could use an ordinary `fetch` request to read the contents of your mail in a neighboring tab — the browser would still send your own cookies there regardless. One important nuance: the restriction is specifically on *reading* the response, not on the ability to send the request at all, which is why `<img src="some-other-site">` works just fine, but you can't read the body of that response from JavaScript.

CORS (Cross-Origin Resource Sharing) is a mechanism by which a server can deliberately relax this rule for itself: in its response to a request it adds an `Access-Control-Allow-Origin: https://example.com` header (or `*`), telling the browser that this origin is allowed to read its responses. The crucial detail here is *who* grants the permission — the server does, not the client. "Disabling CORS" from the page's side is impossible in principle; the only option is to arrange the corresponding setting with whoever owns the server.

When a page is opened over the `file://` protocol (for example, `file:///home/.../index.html`), the browser treats every file on disk as a separate, opaque origin: from the browser's point of view, `file://a.js`, requested from `file://index.html`, shares no common origin with the requesting page, and the module request gets blocked (`Cross-Origin Request Blocked` in the console). This isn't a quirk of one particular implementation, but a consistent restriction shared across every modern browser.

Per the project's functional requirements — no server side, and it has to open directly from disk, with no local server: handing a colleague a working copy means handing over a folder of files, with no `npm start` and no external dependency of any kind. Since `import`/`export` are unavailable under these conditions, there's exactly one working option left — the classic `<script src="...">` injection, without a `type="module"` attribute. Such tags load as ordinary files rather than as modules, and are not subject to CORS rules at all.

### The global object as a stand-in for the import mechanism

Giving up `import`/`export` also means giving up the automatic mechanism by which they pass functions and variables between files. The solution implemented in the project is exactly the one the whole of JavaScript used before modules existed, back in the jQuery-plugin era, roughly 2005-2015: one shared object in global scope, which every file writes into whatever it wants to expose, and which later-loaded files read from.

There are two such objects in the project, split by purpose: `window.DS` — the namespace for data (sources that get loaded by separate `<script>` tags while the app is already running, not at startup), and `window.DS_APP` — the namespace for code, meaning the functions of each module.

This scheme directly explains why the order of `<script>` tags in `index.html` is critical: if `view-build.js`, at the moment it runs, reaches for `App.el` — a function declared in `util.js` — then `util.js` has to appear earlier in the document. There is no automatic dependency-graph construction here, unlike with ES modules: the order is set by hand and matches the order in which the files are physically executed, top to bottom.

### IIFE and closures: privacy without a module system

If every file simply declared `var el = ...` at the top level, all files would share the same global scope — and the name `el` from one file would overwrite the same-named `el` from another, exactly in the spirit of the communal kitchen described above. The solution is to wrap the entire contents of a file in an immediately-invoked function expression, or IIFE: `(function () { ... })()`. Everything declared inside such a function remains its local variables and does not leak into global scope; only what's explicitly written to `window.DS_APP.name = ...` is published outward — the same split between private and public that `export` provides in modules, only built by hand rather than by the language.

This trick is directly tied to the mechanism of closures, and the connection is worth spelling out explicitly here. The IIFE creates a scope, but a closure is the mechanism that keeps that scope from disappearing once the function has already finished running. Without closures, the IIFE trick itself would be pointless: the function would run and immediately lose all its variables, and there would be no way to hide anything in it for long.

The general definition is this: a closure is a function together with the scope it captured — the very scope in which that function was created. As long as even one live reference to the function exists, every variable it captured stays alive too — even if the outer function that declared those variables returned control long ago and would seem to have finished its work.

Let's look at this mechanism on a real fragment:

```js
(function (root) {
  var DEFAULT_COL_W = 120;                 // private variable of the IIFE
  var colWidthPx = function (state) {      // function created inside this scope
    return ... DEFAULT_COL_W ...;          // and reading DEFAULT_COL_W
  };
  root.DS_APP.colWidthPx = colWidthPx;     // a reference to the function goes out, onto window
})(window);
// The IIFE has run and finished. But DEFAULT_COL_W keeps existing:
// the closure inside colWidthPx holds a reference to it, and colWidthPx itself lives on window.DS_APP.
```

From outside, `DEFAULT_COL_W` is unreachable in the fullest sense of the word: there is no syntax that could reach it from another file or from the developer console. At the same time, the variable doesn't stop existing — it holds a value and is accessible to exactly the one function that needs it. This is encapsulation in its strictest form: not merely a "don't touch" convention at the level of a contract, but the physical inaccessibility of the variable to the developer. As a contrast, recall the `_`-prefix convention that appears in the same project (`_entityColumns`, `_swap`) — there, privacy is only signaled by name, and nothing stops you from calling `DS_APP._swap` directly, if the function happens to be exported anyway.

The same mechanics get used in languages where there's no real need for it. Python supports closures, and historically they've been used for encapsulation in exactly the same way:

```python
def make_counter():
    count = 0                  # local variable of the outer function
    def increment():
        nonlocal count         # explicit marker: write to the outer variable, don't create your own
        count += 1
        return count
    return increment           # count stays alive in the returned function's closure

c = make_counter()
c()   # 1
c()   # 2
# count cannot be reached from outside: c.count doesn't exist, it's not an object attribute
```

The difference between JavaScript and Python is in how necessary this trick actually is. In Python, closures are almost never needed for encapsulation: it has a real module system (`import` works without bundlers and without any CORS-like analog) and classes, with privacy inside them governed by convention (`_name` as an internal-by-agreement name, double underscore as a name-mangling mechanism — itself still bypassable, though). A Python programmer is far more likely to reach for a closure to build a factory function, a decorator, or partial application than simply to hide a constant — for that, it's enough to put it in a module and not export it.

In Rust, the discipline is even stricter: a name's visibility is determined by the explicit `pub` keyword (or its absence), and the compiler leaves no way around that decision — we'll come back to Rust's module system in the section devoted to it in the Rust track below. Closures exist in Rust too (the `Fn`, `FnMut`, `FnOnce` traits), but they aren't given the role of a privacy mechanism — that role belongs entirely to the language's module system.

JavaScript, before ES modules appeared, turned out to be the only one of the three languages discussed here with no module system at all — which is why a closure wasn't just one of several encapsulation tools available to it, but the only one. The IIFE you can find at the top of every file in the project isn't a stylistic quirk, but exactly the solution the entire JavaScript community adopted during 2005-2015, and for the exact same reason — there simply was no alternative.

Subtler aspects of how closures work — in particular, the trickiest one of all: a closure holds onto a variable, not the value it had at creation time — are covered in detail in chapter 7, using a live example of the `currentState`/`currentDispatch` pair from `view-table.js`.

### Further reading:

- MDN, **JavaScript modules** — the main reference on ESM:
  https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules
- MDN, **`<script>`** (the `type="module"`, `defer`, `async` attributes):
  https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script
- MDN, **Same-origin policy** — what an origin is and exactly what's forbidden:
  https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy
- MDN, **CORS**:
  https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
- MDN, **Closures** — closures from scratch, with encapsulation examples:
  https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Closures
- MDN, **IIFE** (glossary):
  https://developer.mozilla.org/en-US/docs/Glossary/IIFE
- **The Rust Book, ch. 7** "Managing Growing Projects with Packages, Crates,
  and Modules" — the point of comparison for the Rust track below; especially the
  "Defining Modules to Control Scope and Privacy" and "Paths for Referring to an
  Item in the Module Tree" subsections:
  https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html
- **rustlings**, the `modules` section — exercises for the same chapter:
  https://github.com/rust-lang/rustlings
- Python, **Scopes and Namespaces** — for comparison, how it's done there:
  https://docs.python.org/3/tutorial/classes.html#python-scopes-and-namespaces

If you want some historical context (why JS lived without modules for so long,
and what came before ESM — CommonJS, AMD/RequireJS, UMD): look up the **UMD**
format — the project effectively uses a simplified version of it, and in
chapter 4 you'll see this explicitly, when the same file turns out to be both
a browser script and a Node module for tests, at the same time.

## Code

## Practice

## DevTools

## Rust

## Self-Check
