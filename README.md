# @chuks/textmate

A **TextMate grammar interpreter written 100% in Chuks** — tokenizes source code
into scoped spans using `.tmLanguage.json` grammars, the same grammars VS Code and
Shiki use. It's the server-side syntax-highlighting engine for Chuks: no Shiki, no
oniguruma WASM blob, no Node.

It runs on [`@chuks/pcre`](https://github.com/chuks-lang/chuks_pcre) — the
backtracking regex engine that gives grammars the lookaround, backreferences and
`\G` anchor they depend on. Pure Chuks end to end, identical on the VM and the AOT
native backend.

## Install

```sh
chuks add @chuks/textmate
```

(`@chuks/pcre` is pulled in automatically as a dependency.)

## Quick start

```chuks
import { Grammar, GrammarState, LineTokens, Token } from "pkg/@chuks/textmate"

var g = Grammar.load(grammarJsonString)      // a .tmLanguage.json string
var state = g.initialState()

for (var line of sourceLines) {
    var result = g.tokenizeLine(line, state)
    for (var t of result.tokens) {
        // t.start, t.end  → rune offsets in the line
        // t.scopes        → scope stack, e.g. ["source.js","string.quoted.double"]
    }
    state = result.state                     // carry multiline context to next line
}
```

Each token's `scopes` is the full stack from the grammar's root scope down to the
most specific. Map the **last** scope (or the whole stack) to a color through a
theme to render highlighted code.

## Supported grammar constructs

| Construct                | Supported |
|--------------------------|-----------|
| `match` rules            | ✅ with numbered `captures` → sub-scopes |
| `begin` / `end` rules    | ✅ with `beginCaptures`, `endCaptures`, `contentName`, nested `patterns` |
| `include`                | ✅ `$self`, `$base`, `#repository-ref` |
| `repository`             | ✅ |
| Multiline regions        | ✅ state carried across lines via `GrammarState` |
| Backreference `end`      | ✅ `\1`-style heredoc delimiters substituted from the `begin` match |
| Scope stacks             | ✅ accumulated root → leaf per token |

Not yet supported: `while` rules, injection grammars, and cross-grammar
`include`s of external scopes (e.g. embedding `source.css` inside HTML) — these
are on the roadmap.

## API

### `class Grammar`

```chuks
Grammar.load(jsonStr: string): Grammar     // parse a .tmLanguage.json string
g.initialState(): GrammarState             // fresh per-document state
g.tokenizeLine(line: string, state: GrammarState): LineTokens
```

### `class LineTokens`

```chuks
result.tokens: []Token        // left-to-right, covering the whole line
result.state: GrammarState    // pass into the next line's tokenizeLine
```

### `class Token`

```chuks
t.start: int       // start rune offset
t.end: int         // end rune offset
t.scopes: []string // scope stack, root → most specific
```

### `class GrammarState`

Opaque multiline context (the active rule stack). Create one with
`g.initialState()` per document, thread the `result.state` from each line into the
next. `state.clone()` is available if you need to fork it.

## How it works

`Grammar.load` parses the grammar JSON into a tree of compiled rules (each `match` /
`begin` / `end` regex is compiled once via `@chuks/pcre`). `tokenizeLine` runs the
classic TextMate loop: at each position it finds the earliest-matching pattern among
the active rule's children (plus the current region's `end`), emits the text before
it in the current scope, then either emits a captured match or pushes/pops a
begin/end region. The rule stack persists in `GrammarState` so a string or block
comment opened on one line stays open on the next.

## Tests

```sh
chuks run tests/index.test.chuks                          # VM
chuks build tests/index.test.chuks -o /tmp/t && /tmp/t    # AOT
```

21 assertions — match rules, captures, includes, multiline block comments,
heredoc backreference delimiters, and scope stacks — green on both the bytecode VM
and the AOT native backend.

## License

MIT
