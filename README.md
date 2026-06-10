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
most specific. To go straight to colors, use a `Theme` + `Highlighter` (below)
instead of `tokenizeLine` — a complete syntax highlighter, pure Chuks.

## Full highlighting (grammar + theme)

```chuks
import { Grammar, Theme, Highlighter } from "pkg/@chuks/textmate"

var hl = new Highlighter(Grammar.load(grammarJson), Theme.load(themeJson))
var state = hl.initialState()
for (var line of sourceLines) {
    var styled = hl.highlightLine(line, state)
    for (var t of styled.tokens) {
        // t.text, t.foreground ("#rrggbb"), t.bold, t.italic
    }
    state = styled.state
}
```

`Theme.load` accepts a VS Code / TextMate theme JSON (the shape Shiki ships).
`Theme.resolve(scopes)` returns a `Style { foreground, bold, italic, underline }`
using TextMate scope-selector specificity (descendant selectors, scope arrays,
comma lists, the no-scope default).

## Embedded languages

Use a `GrammarRegistry` when one grammar embeds another (HTML's `<style>` →
`source.css`, markdown fences → `source.js`, …):

```chuks
import { GrammarRegistry } from "pkg/@chuks/textmate"

var reg = new GrammarRegistry()
reg.load(cssGrammarJson)              // register source.css
var html = reg.load(htmlGrammarJson)  // its `{include:"source.css"}` now resolves
var r = html.tokenizeLine("color: red", state)
```

## Supported grammar constructs

| Construct                | Supported |
|--------------------------|-----------|
| `match` rules            | ✅ with numbered `captures` → sub-scopes |
| `begin` / `end` rules    | ✅ with `beginCaptures`, `endCaptures`, `contentName`, nested `patterns` |
| `include`                | ✅ `$self`, `$base`, `#repository-ref`, and external `source.x` / `source.x#rule` via `GrammarRegistry` |
| `repository`             | ✅ |
| Embedded languages       | ✅ cross-grammar includes through `GrammarRegistry` |
| Multiline regions        | ✅ state carried across lines via `GrammarState` |
| Backreference `end`      | ✅ `\1`-style heredoc delimiters substituted from the `begin` match |
| Scope stacks             | ✅ accumulated root → leaf per token |
| Theme resolution         | ✅ VS Code themes → per-token color + font style |

Robust by design: a rule whose regex uses an oniguruma feature `@chuks/pcre`
doesn't support is skipped (the grammar still loads), the `end` pattern has
TextMate's default priority, and `captures` accept both the `{name}` and bare
string forms. Validated against the real Shiki grammars for 21 languages
(TypeScript, JS, Python, Go, Rust, Java, C/C++, C#, Ruby, PHP, Swift, Kotlin,
HTML, CSS, JSON, YAML, Bash, SQL, Markdown, Chuks).

Not yet supported: `while` rules and injection grammars — on the roadmap.

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

### `class Theme`

```chuks
Theme.load(jsonStr: string): Theme       // VS Code / TextMate theme JSON
theme.resolve(scopes: []string): Style   // best style for a scope stack
theme.defaultForeground(): string        // editor foreground
theme.background(): string               // editor background
```

### `class Style`

```chuks
style.foreground: string   // "#rrggbb" ("" if unset)
style.bold: bool
style.italic: bool
style.underline: bool
```

### `class Highlighter`

```chuks
new Highlighter(grammar: Grammar, theme: Theme)
hl.initialState(): GrammarState
hl.highlightLine(line, state): StyledLine   // .tokens: []StyledToken, .state
hl.defaultForeground(): string
hl.background(): string
```

`StyledToken` carries `start`, `end`, `text`, `foreground`, `bold`, `italic`.

### `class GrammarRegistry`

```chuks
new GrammarRegistry()
reg.load(jsonStr: string): Grammar    // parse + register (keyed by scopeName)
reg.add(grammar: Grammar): void
reg.get(scopeName: string): Grammar?
reg.has(scopeName: string): bool
```

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

44 assertions across `tests/index.test.chuks` (grammar), `tests/theme.test.chuks`
(theme + highlighter) and `tests/registry.test.chuks` (embedded languages) —
green on both the bytecode VM and the AOT native backend.

## License

MIT
