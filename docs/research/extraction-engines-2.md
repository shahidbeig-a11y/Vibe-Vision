# Research: deterministic language-agnostic structural extraction

Resolves [#2](https://github.com/shahidbeig-a11y/Vibe-Vision/issues/2). Feeds the stack decision ([#8]) and the extraction spike ([#10]).

## Question

What engine gives deterministic, language-agnostic extraction of code structure good enough to power (a) an architecture zoom (modules, dependencies, entry points) and (b) evidence anchors under feature verdicts (routes, functions, tests, symbols)?

## Recommendation

**tree-sitter as the parsing core, with a thin per-language query layer on top.** No single tool covers both layers deterministically across languages — dependency-cruiser is JS/TS-only, pydeps and `go list` are single-ecosystem, and stack-graphs/SCIP solve cross-file *resolution* (a harder, narrower problem) at the cost of per-language rule-writing effort. tree-sitter is the one primitive that is genuinely language-agnostic, has official grammars for the top vibe-coder languages (TS/JS, Python, Go, Rust, etc.), parses standalone files without a build step, and has a native incremental re-parse API built for exactly the live-file-watch case Vibe-Vision needs. Build the architecture zoom and evidence anchors on tree-sitter queries (function/route/import/test-symbol patterns per language); reserve cross-file symbol resolution (stack-graphs/SCIP) as a v2 upgrade only if flat per-file extraction proves insufficient for "which module calls which."

## Engines surveyed

### tree-sitter — the parsing core

- **What it is**: "An incremental parsing system for programming tools" that builds a concrete syntax tree per source file and efficiently updates it as the file is edited. Aims to be general enough for any language, fast enough to run per-keystroke, and robust to syntax errors. [tree-sitter/tree-sitter](https://github.com/tree-sitter/tree-sitter)
- **Determinism**: Parsing is a pure function of grammar + source text — no build step, no environment, no network. This is the property no other tool here fully has.
- **Incremental re-parse**: The parse function (`ts_parser_parse` in C, exposed via bindings) accepts an `old_tree` argument; when combined with an edit description (`tree.edit()` / `InputEdit`: start byte, old end byte, new end byte, plus positions), tree-sitter reuses unaffected subtrees and only re-parses the changed region. Official docs confirm: "If the source code changes, Tree-sitter updates only the affected parts of the tree, which saves time and memory." [Using Parsers](https://tree-sitter.github.io/tree-sitter/using-parsers/) — **this is a direct fit for live file-watch updates**, not a workaround.
- **Extraction mechanism**: The query language (S-expressions matching AST shapes, e.g. `(function_definition name: (identifier) @name) @definition.function`) lets you capture named nodes — function defs, imports, class defs — with field-name-aware patterns and predicates (`#eq?`, `#match?`). [Tree-sitter queries](https://tree-sitter.github.io/tree-sitter/4-code-navigation.html)
- **Language coverage**: 23 languages maintained directly under the `tree-sitter` GitHub org; community packs (`tree-sitter-language-pack`) bundle 300+ grammars with polyglot bindings (Rust, Python, Node.js, Go, Java, etc.). TS/JS, Python, Go, Rust, Java, Ruby, PHP, C/C++ all have mature, widely-used grammars.
- **Rust bindings**: `tree-sitter` crate on crates.io (0.26.x as of this writing) exposes `Query`, `QueryCursor`, `QueryCapture` — the same query API as other bindings, callable from a Rust host process. [docs.rs/tree-sitter](https://docs.rs/tree-sitter)
- **What it explicitly does NOT do**: no semantic analysis, no symbol resolution across files, no type information, no understanding of "this import binds to that definition." It gives you a syntax tree per file — nothing more. This is the deterministic floor's ceiling.

### dependency-cruiser (v18) — JS/TS dependency graph, confirmed ecosystem-bound

- JS/TS-focused; also handles CoffeeScript, LiveScript, Vue SFCs, and Svelte **if the relevant transpiler is installed** — not standalone/deterministic across environments. [dependency-cruiser FAQ](https://github.com/sverweij/dependency-cruiser/blob/main/doc/faq.md)
- Parses via Acorn (+ acorn-jsx) or the TypeScript compiler when available, with SWC as an optional faster path — i.e., it delegates to a real language parser/compiler, not a lightweight text scan.
- v18.0.0 explicitly fixed a bug to make "the summary more deterministic," and now requires Node ^22/^24/≥26. v18.1.0 adds environment-consistency checks (warns if a needed compiler like TS/Babel/SWC isn't available). [Releases](https://github.com/sverweij/dependency-cruiser/releases)
- Known gap: dynamic `require`/`import` with variables/expressions can't be resolved — only static strings and template literals.
- **Verdict**: excellent, mature JS/TS module-graph tool, but it is not the language-agnostic layer — it's evidence that a single polished per-ecosystem tool doesn't generalize, reinforcing the case for a shared tree-sitter core with per-language query packs instead of chasing an equivalent tool per ecosystem.

### pydeps — Python-only, bytecode-based, confirmed lossy on dynamic imports

- Finds imports by scanning import opcodes in compiled bytecode, not source-level static analysis. [thebjorn/pydeps](https://github.com/thebjorn/pydeps)
- Only follows imports it can actually resolve through Python's import machinery; unresolvable/missing modules are dropped unless `--include-missing` is passed.
- Confirmed limitation: `from module import *` on an extension module can't be resolved without actually importing it — an explicit non-determinism/incompleteness admission from the maintainers.
- **Verdict**: single-ecosystem, and even within Python it's not a clean deterministic parse — it depends on bytecode compilation and import resolution, which can vary by environment. Not a foundation piece; at most a cross-check.

### `go list` — Go-only, requires the Go toolchain and module resolution

- `go list -json -deps` gives structured package/dependency data, but it shells out to the real Go toolchain and module resolver — this is toolchain-dependent, not a standalone deterministic parse. [go list docs](https://pkg.go.dev/cmd/go/internal/list)
- Determinism of `go list -m` output has itself been an open discussion in the Go issue tracker (issue #28019) — even the canonical tool has had non-determinism bugs.
- **Verdict**: fine as a Go-specific evidence source later, but it's a toolchain integration, not part of a language-agnostic core.

### stack-graphs (GitHub) — deterministic, but solves a narrower/harder problem

- Purpose-built for **name resolution and code navigation** (go-to-definition, find-references) across and within files — powers GitHub's Precise Code Navigation. [Introducing stack graphs](https://github.blog/open-source/introducing-stack-graphs/)
- Explicitly deterministic and incremental by design: graph construction is file-incremental (each file's subgraph is built with no visibility into other files), stored in SQLite, and resolved via a path-stitching search algorithm — no build tools required.
- Built on tree-sitter: language-specific TSG (Tree-Sitter-Graph) rules transform the AST into the stack graph.
- **The catch**: each language needs its own hand-written TSG rule set defining that language's name-binding semantics (scoping, imports, shadowing). This is real engineering work per language — it's the layer above tree-sitter that gives you *symbol resolution*, not a drop-in.
- **Verdict**: the right tool if/when Vibe-Vision needs true cross-file "who calls whom" resolution rather than per-file structural facts. Not needed for v1 architecture zoom or evidence anchors, which can be satisfied by per-file extraction plus import-graph stitching.

### SCIP / LSIF — protocol-level, not extraction engines themselves

- **LSIF** (Microsoft): a dump format for a workspace's language-server knowledge, JSON-based, modeled on LSP types. Explicit stated limitation: "LSIF doesn't contain any program symbol information nor does it define any symbol semantics" — it's a cache of LSP *answers*, not a symbol database. [LSIF spec](https://microsoft.github.io/language-server-protocol/specifications/lsif/0.6.0/specification/)
- **SCIP** (Sourcegraph): the LSIF successor — a protobuf-based, language-agnostic protocol, explicitly designed to be easier to produce/debug than LSIF. [SCIP repo](https://github.com/sourcegraph/scip)
- Indexers exist per language (scip-typescript, scip-python, scip-java, rust-analyzer emits SCIP directly, scip-clang, scip-ruby, scip-dotnet, scip-dart, scip-php) — but **each indexer is effectively a compiler/language-server integration**, meaning it needs a working build/type-check environment for that project (e.g., scip-python leans on Pyright-style analysis, scip-java needs a working Maven/Gradle build). That is the opposite of "deterministic parse of arbitrary vibe-coder code that may not even build."
- **Verdict**: SCIP is the right *interchange format* if Vibe-Vision ever needs to ingest or emit cross-tool code-intelligence data, but it is not itself an extraction engine, and its indexers assume a buildable project — a shakier assumption for AI-agent-generated codebases than "the files parse as valid syntax."

### ast-grep — tree-sitter-based CLI, same floor/ceiling as tree-sitter itself

- Confirmed built directly on tree-sitter grammars; CLI exposes structural search/rewrite/lint via YAML-configured AST patterns (`$MATCH`-style wildcards) across ~25 languages out of the box, more via custom grammar registration. [ast-grep.github.io](https://ast-grep.github.io/), [ast-grep/ast-grep](https://github.com/ast-grep/ast-grep)
- Effectively a batteries-included wrapper around exactly the tree-sitter query/pattern-matching primitive Vibe-Vision would use directly — good candidate for authoring/testing extraction patterns quickly (e.g., prototyping "find all Express routes" as an ast-grep rule) before committing to hand-written tree-sitter queries in the actual engine, but doesn't add a capability tree-sitter itself lacks (no cross-file resolution, no semantic layer).

## Where the deterministic floor ends across top vibe-coder languages

| Layer | TS/JS | Python | Go | Rust | Others w/ tree-sitter grammar |
|---|---|---|---|---|---|
| Syntax tree (functions, imports, classes, decorators) | tree-sitter — deterministic | tree-sitter — deterministic | tree-sitter — deterministic | tree-sitter — deterministic | deterministic if a maintained grammar exists |
| Per-file symbol declarations (routes, exported functions, test names) | tree-sitter query — deterministic | tree-sitter query — deterministic | tree-sitter query — deterministic | tree-sitter query — deterministic | same |
| Same-file import statements (raw strings) | tree-sitter — deterministic | tree-sitter — deterministic | tree-sitter — deterministic | tree-sitter — deterministic | same |
| Resolving an import string to the actual file/module it points to | needs module-resolution logic (like dependency-cruiser's) — deterministic but ecosystem-specific rules (tsconfig paths, node_modules, aliases) | needs Python import-resolution rules (relative imports, `sys.path`, packages) | `go list`/module resolution — needs the Go toolchain | Cargo-aware resolution | ecosystem-specific every time |
| Cross-file symbol resolution ("this call site binds to that definition") | stack-graphs / SCIP, or an agent reading both ends | same | same | same | same |
| Runtime/dynamic behavior (dynamic imports, reflection, string-built routes, `eval`) | **floor ends here** — no deterministic tool resolves these | same | same | same | same |

Floor summary: **tree-sitter's per-file AST + query extraction is deterministic and language-agnostic for the first three rows across every mainstream vibe-coder language.** Import-string resolution to real files is deterministic but requires a small ecosystem-specific resolver per language (this is the extra work in ticket #10, not a research gap — dependency-cruiser and `go list` show the pattern to copy). Cross-file resolution and anything dynamic/runtime is where a human or an agent has to take over — no engine surveyed here gives that deterministically for code that may not even compile/build (a realistic case for AI-agent-generated repos).

## Incremental re-parse for live file-watch

tree-sitter's edit API (`tree.edit()` with an `InputEdit` describing the byte range changed, then re-parsing with the old tree passed in) is purpose-built for the "user/agent just changed one file, update the map" scenario — it reuses unaffected subtrees rather than re-parsing whole files, and it's fast enough to run per-keystroke in editors, so per-file-save is comfortably inside its design envelope. Practically: keep one parser + one tree per file in memory, feed it edits as they land from a file watcher, and only affected files need re-parsing — cross-file graph edges (imports/dependencies) get recomputed for the changed file's edges only, not the whole repo.

## Facts most load-bearing for #8 (stack decision) and #10 (extraction spike)

1. **tree-sitter is the only surveyed engine that is simultaneously deterministic, language-agnostic, standalone (no build/compile step), and built for incremental re-parse** — every other tool is either single-ecosystem (dependency-cruiser, pydeps, `go list`) or requires a working build/type-check environment per language (SCIP indexers), which AI-agent-generated code can't reliably guarantee.
2. **The deterministic floor covers per-file structure (functions, imports, classes, routes, tests) but stops at cross-file symbol resolution and anything dynamic** — resolving "this import string points to that file" needs a small per-ecosystem resolver (cheap, same pattern dependency-cruiser/`go list` already use), while "this call site invokes that other module's function" needs stack-graphs-style per-language rules or an agent — that's the real scope boundary for ticket #10's spike.
3. **tree-sitter's edit/re-parse API (old-tree reuse via `InputEdit`) is a direct, purpose-built fit for live file-watch updates**, not an adaptation — no custom incremental-parsing logic needs to be invented for #10.
