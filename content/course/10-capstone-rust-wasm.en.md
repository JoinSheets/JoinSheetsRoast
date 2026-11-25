+++
title = "Capstone: formula.js → Rust + wasm-bindgen → WASM"
weight = 10
+++

Code: all of `formula.js` (356 lines) — `tokenize`, `makeParser`/the AST,
`evaluate`. `tests/formula.test.js` is the specification of the expected
behavior, which you'll need to reproduce in Rust. This is the capstone
chapter: not finished code to walk through, but a plan for porting part of
the code from JS to Rust, which you're meant to implement yourself.

## Theory

### WebAssembly as yet another virtual machine in the browser

WebAssembly isn't "JS, but compiled" — it's a separate, compact, typed binary bytecode format that the browser runs in its own sandboxed VM, alongside the regular JS engine, not in place of it. Code for WASM is usually written in a language that has a compiler targeting WASM — Rust, C/C++, Go, and others — and the resulting module is nothing like a "subset of JS": it has no DOM of its own, no `document` of its own, no garbage collector in the familiar sense. It's a low-level virtual machine with linear memory — one big array of bytes — and a set of exported functions, nothing more. `wasm-bindgen` is a tool that generates the "glue": a JS file able to call functions in the compiled `.wasm` module and pass values in both directions — numbers almost for free, directly, while strings and objects get copied through the WASM module's linear memory, and that copying isn't free.

### The boundary between languages as a general problem, not a WASM-specific one

The task of "passing control and data from one runtime to another, where the two have different memory models" doesn't only come up at the JS/WASM boundary — it's a special case of a much more general topic, known for decades under the name FFI, foreign function interface. Python code calling a C library through `ctypes` or `cffi`; Java calling native code through JNI (Java Native Interface); Node.js attaching C++ add-ons through N-API — in every one of these cases, every crossing of the boundary costs noticeably more than an ordinary function call within a single runtime: data representations have to be converted, memory has to be copied, sometimes the garbage collectors of two different runtimes have to be synchronized. The general engineering lesson here is the same regardless of the specific pair of languages: you should cross the boundary rarely, with large chunks of data, rather than often and in small pieces — the cost of a single crossing is roughly constant, and the more useful work you can get done in one crossing, the smaller that cost is per unit of result.

This general lesson directly determines how `formula.js` needs to be ported to Rust+WASM — as opposed to the most common and, at first glance, most natural mistake of the "let's rewrite it in Rust+WASM to make it faster" approach: porting the computation over in exactly the way it's called right now in JS, without thinking about the boundary between the two runtimes at all. `formula.js::evaluate(ast, ctx)`, in `JoinSheets`'s real pipeline (`entities.js::resolveCell`), gets called once per cell, for every row that has a formula — for a table with 10,000 rows and 3 formula columns, that's 30,000 separate calls. If each such call is naively ported across the JS→WASM boundary one at a time (passing the formula and the context, getting back a result string), the overhead of 30,000 separate boundary crossings is perfectly capable of eating up the whole gain from the faster computation inside. A correct port has to cross the boundary less often, but with large chunks of data at once — for example, passing the formula and an entire column of rows together, computing all 10,000 values inside Rust in one call, and returning the result as a single array. This isn't an advanced nuance — it's the decision that determines whether the whole capstone succeeds: without it there's a real risk of ending up with an honest, but disheartening, result of "the WASM version turned out to be slower" — valuable as a finding in its own right, not a failure; but it's avoidable if the boundary is designed correctly from the start.

### Zero-build as an invariant the capstone has to preserve

The same requirement runs through the entire course program for `JoinSheets`: no external dependencies, no build step, no installation — the project has to stay a static set of files, with not a single line of server-side code, that can be opened straight from disk. Any change this chapter proposes has to preserve that requirement, not carve out a special exception "for performance."

Porting part of the code to Rust and compiling it to WASM looks, at first glance, like a step away from this principle: Rust does need to be compiled, and there's no compiler on the end user's machine, nor is there meant to be one. There's no real contradiction here, though, because it's the developer's job to compile the `.wasm` module — once, ahead of time, on their own machine — not the user's, every time they run the app: `wasm-pack build` turns Rust code into a pair of files (a `.wasm` file and a JS "glue" file), and both get committed to the repository as finished, unchanging binary artifacts — exactly the same way `import/vendor/xlsx.full.min.js`, a third-party library also built ahead of time by someone else, is already wired into the project. The user doesn't run `cargo` or `wasm-pack` at all — they get one more `<script src="...">`, included and loaded exactly like any other file in the project, and from their point of view no different from one.

The WASM module, then, isn't an exception to the "no external dependencies" rule, but a narrow, static artifact — it solves one specific performance problem for one specific function, and changes nothing about how the project is distributed and run. It's with this constraint in mind that you should approach the port, the plan for which is laid out below.

### Further reading:

- Wikipedia, **Foreign function interface** — the general concept, of which
  the JS↔WASM boundary is a special case:
  https://en.wikipedia.org/wiki/Foreign_function_interface
- **The Rust and WebAssembly Book** — the main, systematic source for the
  whole capstone; start here, not with individual recipes:
  https://rustwasm.github.io/docs/book/
- **wasm-bindgen Guide** — the reference on the `#[wasm_bindgen]` attribute,
  passing types across the boundary, working with strings and `JsValue`:
  https://rustwasm.github.io/wasm-bindgen/
- **wasm-bindgen, "Supported Types"** — critical for step 10.4: what gets
  passed cheaply, and what gets copied:
  https://rustwasm.github.io/wasm-bindgen/reference/types.html
- **wasm-pack** — the build step and its flags (`--target web`, `--dev`):
  https://rustwasm.github.io/wasm-pack/book/
- MDN, **WebAssembly** — what it is from the browser's side, the concepts of
  module/memory/instance:
  https://developer.mozilla.org/en-US/docs/WebAssembly
- MDN, **WebAssembly.instantiateStreaming** — how the module actually ends up
  on the page (what's under the hood of the JS glue `wasm-pack` generates):
  https://developer.mozilla.org/en-US/docs/WebAssembly/JavaScript_interface/instantiateStreaming
- **serde-wasm-bindgen** — if the JSON string at the boundary turns out to be
  the bottleneck (see the Practice section on measurements):
  https://github.com/RReverser/serde-wasm-bindgen
- **The Rust Book, ch. 6** "Enums and Pattern Matching" — the basis for the
  AST:
  https://doc.rust-lang.org/book/ch06-00-enums.html
- **The Rust Book, ch. 15.1** "Box\<T\>" — why a recursive `enum Ast` requires
  a `Box`:
  https://doc.rust-lang.org/book/ch15-01-box.html
- **The Rust Book, ch. 9** "Error Handling" — `Result<T, E>` instead of JS's
  `throw`/`{ok: false, error}`:
  https://doc.rust-lang.org/book/ch09-00-error-handling.html
- **Crafting Interpreters** (Robert Nystrom) — if you want to go deeper into
  parsing theory than this capstone strictly needs; the scanner and
  recursive-descent chapters map directly onto `tokenize`/`makeParser`:
  https://craftinginterpreters.com/scanning.html
- **Chrome DevTools, debugging WebAssembly**:
  https://developer.chrome.com/docs/devtools/wasm

## Rust

### Step 10.1 — the tokenizer, pure Rust, no WASM

Port `tokenize` (`formula.js` lines 104-167) over first — it doesn't depend on
either the AST or the evaluator.

```rust
#[derive(Debug, PartialEq)]
enum Token {
    Number(f64),
    Str(String),
    Bracket(String),
    Ident(String),
    Op(String),
}

fn tokenize(text: &str) -> Result<Vec<Token>, String> {
    // port of formula.js::tokenize's isDigit/isUpper/main while loop
    // JS throw -> Rust Result<_, String>; JS "unterminated string" -> Err("unterminated string (position N)".into())
    todo!()
}
```

Take `tests/formula.test.js` — find 3-4 tests there specifically about
tokenization/syntax errors (an unterminated string, an unterminated `[`, an
unexpected character) — rewrite them as `#[test]` functions in Rust, and get
them passing under `cargo test`, without a SINGLE thought about WASM at this
step.

### Step 10.2 — the AST and the parser

`formula.js`'s AST nodes (`{type:'binop',op,left,right}`, `{type:'if',cond,
then,else}`, ...) are tagged objects — exactly what an `enum` expresses
naturally in Rust:

```rust
enum Ast {
    Num(f64),
    Str(String),
    Raw(String),                      // [Source.Field]
    Var(String),                      // a letter: A, B, ...
    Neg(Box<Ast>),
    BinOp { op: char, left: Box<Ast>, right: Box<Ast> },
    Cmp { op: String, left: Box<Ast>, right: Box<Ast> },
    If { cond: Box<Ast>, then_: Box<Ast>, else_: Box<Ast> },
    Call { name: String, args: Vec<Ast> },
}
```

(`Box<Ast>` is necessary because `Ast` recursively contains itself; Rust
needs to know a type's size ahead of time, and a recursive `enum` without
`Box` would have a potentially unbounded size — one more lesson in "what JS
does implicitly, Rust requires explicitly.") Port `parsePrimary` through
`parseTernary` (recursive descent, `formula.js` lines 199-291) almost 1:1 —
each JS function becomes a Rust function, and `fail(msg, tok)` (a throw)
becomes `return Err(...)`. Again — bring a good number more tests over from
`tests/formula.test.js` as `cargo test` cases, this time ones covering valid
parses (`IF(A>0, "yes", "no")`, nested parentheses, function calls).

### Step 10.3 — the evaluator

`evaluate(ast, ctx)` (lines 313-348) — a recursive walk of the `enum Ast`
through `match` — should feel more natural than JS's `switch (node.type)`,
precisely because every `Ast` variant already carries its fields in a typed
way (no `node.left`/`node.right` with the risk of a typo in a field name —
Rust won't compile the code if you mix up a field name or forget one of the
`enum`'s variants in a `match`).

### Step 10.4 — `wasm-bindgen`, your first real WASM

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn evaluate_column(formula: &str, known_letters: Vec<String>, rows_json: &str) -> String {
    // rows_json: a JSON array of objects {letter: value, ...} for ALL rows at once —
    // the boundary is crossed ONCE for the whole column, not once per cell (see the theory above)
    todo!()
}
```

`wasm-pack build --target web` → gives you `pkg/your_crate.js` +
`pkg/your_crate_bg.wasm`. Test them FIRST in a throwaway test HTML page (not in
`JoinSheets`) — just create a `<script type="module">import init,
{evaluate_column} from './pkg/your_crate.js'; ...` — make sure a call from a
real browser actually works and returns what you expect.

### Step 10.5 (stretch) — real integration into `JoinSheets`

Follow the same pattern as `import/` (an optional submodule/vendored files, a
`window.DS_APP.importXlsx`-style check before use, chapter 1): add
`<script src="wasm/your_crate.js">` to `index.html` (after `formula.js`,
before `entities.js`, which calls it), with a presence check before use and a
fallback to the existing `formula.js::evaluate` if the WASM file didn't load
(`404`, the submodule not initialized — the same graceful-degradation
pattern).

## Practice

Repeat exercise 9.4 (Performance panel, recording, Bottom-Up) TWICE: once on
the pure-JS version (`formula.js::evaluate`, called row by row, as it is now),
once on your WASM version (called in a batch, as in step 10.4), on the same
dataset with enough formula rows for the difference to be measurable. Record
both Self Time numbers from Bottom-Up. Any outcome is honest and valuable:
"N times faster," "practically no difference," or even "slower, because of
the overhead of serializing JSON across the boundary" (in which case, the
natural next question to investigate on your own: what if you pass a typed
`JsValue` instead of a JSON string, via `serde-wasm-bindgen`, skipping JSON
text parsing altogether?).

## DevTools

**Memory panel** — after the WASM module loads, open Sources → find the WASM
module's entry in the file tree (Chrome can display WASM with debug
information if it was built with `--dev`/debug symbols) — you can even set a
breakpoint inside the Rust code, if `wasm-pack build --dev` included DWARF
debug info. This is an optional, exploratory item — WASM debugging in
DevTools is still less mature than for JS; don't be discouraged if it doesn't
go smoothly on the first try.
