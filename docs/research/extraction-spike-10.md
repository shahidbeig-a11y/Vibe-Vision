# Spike 10 — Tree-sitter extraction on a real agent-built repo

**Verdict: the deterministic-extraction bet holds.** Tree-sitter parsed 100% of a real two-ecosystem agent-built repo (marketing-fleet: TypeScript/React cockpit + Python host) in 1.2 seconds, with zero syntax-error nodes, exact line anchors on every spot-checked symbol, and a ~30-line hand-written import resolver per ecosystem. It even caught two genuinely dead imports the agents left behind.

- Target: `/Users/mo_beig/code/marketing-fleet` (read-only)
- Spike code: scratchpad `spike10/` — `extract.js` (192 code lines, both ecosystems), `resolver-ts.js` (29), `resolver-py.js` (31), `test-resolvers.js` (self-checks, all passing)
- Date: 2026-07-31

## Setup: what installs cleanly and what bites

| Package | Version used | Note |
|---|---|---|
| tree-sitter (node binding) | 0.21.1 | pinned down from 0.25.1 |
| tree-sitter-typescript | 0.23.2 | exposes `typescript` + `tsx` grammars |
| tree-sitter-python | 0.21.0 | pinned down from 0.25.0 |

Two real gotchas, both solved, both relevant to the production design:

1. **Grammar version skew.** `npm install tree-sitter tree-sitter-typescript tree-sitter-python` fails out of the box: tree-sitter-typescript@0.23.2 peer-requires tree-sitter ^0.21 while tree-sitter-python@0.25 requires ^0.25. Grammars move at different speeds; the product must pin a tested version matrix, not float on `latest`.
2. **32KB string limit.** `parser.parse(source)` throws `Invalid argument` on files over ~32KB. All 7 initial parse failures (5 TSX, 2 Python — real agent-built files, 33–39KB) were this, not syntax. Fix is one line: pass `{bufferSize: src.length + 1}`. After the fix: 244/244 parsed.

## Full-pass numbers

| Metric | TS + TSX | Python | Total |
|---|---|---|---|
| Files | 166 | 78 | 244 |
| Parsed OK | 166 (100%) | 78 (100%) | 244 (100%) |
| Files with ERROR nodes | 0 | 0 | 0 |
| Functions extracted | 544 | 558 | 1,102 |
| Classes | 1 | 40 | 41 |
| Exports / public symbols | 513 | 456 | 969 |
| Imports | 768 | 407 | 1,175 |

- **Runtime: 1.2s** for the full pass (parse + extract + resolve, ~47K source lines: 31.6K TS, 15.4K Py) on an M-series Mac, single-threaded. ~5ms/file. Extraction cost is effectively free at vibe-coder repo scale.
- Zero ERROR/MISSING nodes across the whole repo — agent-generated code is syntactically clean, which is exactly what makes deterministic extraction viable.

## Import → file resolution

| Outcome | TS + TSX (768) | Python (407) |
|---|---|---|
| Internal (resolved to a repo file) | 372 | 170 |
| External package | 394 | 47 |
| Stdlib | — | 190 |
| Unresolved | 2 | 0 |

- **The 2 TS unresolved are correct findings, not resolver misses**: `cockpit-app/scripts/capture-tier-a-stories.ts:10-11` imports `./helpers/audit` and `./helpers/hostMocks` — the `helpers/` directory does not exist. Dead imports left by an agent, caught deterministically. This is precisely the evidence class Vibe-Vision wants.
- **Known misclassification, quantified**: 140 of the 394 "external" TS imports are `@/...` alias imports inside `archive/worktree-wip/` — stale duplicate copies of the app that agents archived, where the live `cockpit-app` tsconfig alias scope doesn't apply. Treating each archived copy as its own alias root (per-directory tsconfig discovery, ~15 more resolver lines) would resolve them. Effective TS resolution accuracy: 628/768 (82%) as-built, ~100% with tsconfig discovery. Archived duplicate trees are themselves a signature vibe-coder repo trait — the product will meet them constantly.
- Python resolution was 100% classified with zero unresolved: dotted imports probed against the repo root, relative imports walked by dot-level, stdlib split via `sys.stdlib_module_names`, third-party via requirements.txt.

## Spot-check: evidence-anchor quality (5 files per ecosystem)

Method: read each source file, compare against extracted names, line numbers, and import classifications.

| File | Checked | Result |
|---|---|---|
| `cockpit-app/components/cockpit/CockpitShell.tsx` (39KB, bufferSize case) | `CockpitShell:71`, `matchesAgent:326`, 20 imports incl. 12 `@/` alias → internal | Exact |
| `cockpit-app/app/page.tsx` | `HomePage:3` (export default), 1 external import | Exact |
| `cockpit-app/scripts/capture-tier-a-stories.ts` | `repoRoot:13`, `boot:30`, `snap:36`; 2 unresolved imports | Exact; unresolved confirmed dead (no `helpers/` dir) |
| `cockpit-app/lib/fleetApi.ts` | `getBusinessSlug:11`, `setBusinessSlug:16`, `fetchDeployment:28`, `DEFAULT_BUSINESS_SLUG:6`, `DeploymentInfo:20`; 0 imports | Exact — 60 exports and 0 imports both real (pure HTTP-client module) |
| `cockpit-app/test/scheduleEditor.test.ts` | 0 functions, 2 imports (`vitest`, `../lib/scheduleEditor` → internal) | Correct — `describe`/`it` callbacks are anonymous args, deliberately not API symbols |
| `host/app.py` | `_gmail_watch_auto_renew_loop:53`, `_schedule_clock_loop:77`, `_lifespan:103`, `health:149`, `get_deployment:163`; fastapi→external, host.api→internal | Exact; underscore-prefixed correctly excluded from public symbols |
| `host/cron.py` | `set_schedule:14`, `replace_schedules:53`, `fire_schedule:62`; host.settings_model→internal | Exact |
| `tests/test_api_seam.py` (38KB, bufferSize case) | `test_health_no_margn_branding:6`, `test_01_roster...:16`, `test_01_cockpit_html_served:35`; tests.conftest→internal | Exact |
| `tools/registry.py` (34KB) | `register:31`, `get:34`, `call:57`, `hubspot_read:101`; tools.providers→internal | Exact |
| `runtime/bundle.py` | `agents_package_root:34`, `load_bundle:45`; runtime.config→internal (relative + absolute both verified) | Exact, one gap: `from __future__ import annotations` (line 3) missed |

10/10 files: every extracted symbol name and line number matched the source exactly. One extraction gap found (below).

## Noise level

Low. Concretely:

- 0 anonymous functions captured (extractor only names real symbols; test-runner callbacks excluded by construction).
- Noisiest file is `fleetApi.ts` at 60 exports — verified real (a fat hand-rolled API client), not extraction noise. The count itself is a useful code-smell signal.
- One systematic miss: `from __future__ import annotations` parses as a distinct `future_import_statement` node type and my visitor skipped it — 64 Python files affected. Zero impact on dependency graphs (`__future__` is a compiler directive), but it illustrates the real tax: each grammar has a long tail of node types, and coverage is a checklist you build per grammar, not something you get for free.
- Python "exports" are a convention call (top-level non-underscore defs; `__all__` not parsed). Good enough for evidence anchors; the production version should honor `__all__`.

## The resolver tax

| Component | Code lines | What it covers |
|---|---|---|
| TS/TSX resolver | 29 | relative paths, `@/*` tsconfig alias, 7-extension + index probing; bare = external |
| Python resolver | 31 | absolute dotted vs source roots, relative dot-walking, `__init__.py`, stdlib/third-party split |
| Extraction visitors (both ecosystems) | ~150 of extract.js's 192 | functions/classes/exports/imports per grammar |

So the marginal cost of an ecosystem is roughly **~30 lines of resolver + ~75 lines of grammar visitor**, plus the per-grammar node-type checklist. The un-modeled remainder for production: tsconfig discovery per sub-project (~15 lines), `__all__`, `require()`/dynamic import edge cases, monorepo workspace aliases — each an afternoon, not a subsystem.

**How many ecosystems does a typical vibe-coder repo set imply?** This one repo alone had two. Across Mo's own repo set (Herdr, gbrain, Coordinator, marketing-fleet, Wayfinder) the pattern is consistent: **TS/JS + Python covers the overwhelming bulk**, with a long tail of shell/SQL/config that matters far less for symbol-level evidence. Budget for 2 ecosystems at launch, ~100 lines each, with the architecture leaving grammar+resolver as a pluggable pair.

## What this means for the bet

1. **Deterministic extraction works on real agent output.** 100% parse rate, zero grammar errors, exact line anchors — agent-generated code is more syntactically uniform than human code, which favors this approach.
2. **Speed is a non-issue.** Full repo in ~1.2s means extraction can run on every scan, no caching architecture needed at this scale.
3. **The resolver is the real (small) engineering surface.** Parsing is solved; import→file resolution is where per-ecosystem judgment lives, at ~30-100 lines each. The failure modes found (alias scope in duplicated trees, `future_import_statement`) are enumerable and cheap, not open-ended.
4. **Extraction already produces product-grade findings for free**: dead imports to nonexistent files, a 60-export god-module, stale archived app copies — all surfaced deterministically with file:line evidence.

Caveat worth carrying forward: this repo had zero syntax errors, so ERROR-node recovery is untested here. Tree-sitter is designed to keep parsing around errors, but a broken-mid-edit vibe-coder repo should be the next adversarial input if we want that claim proven too.
