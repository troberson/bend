---
name: bend
description: >-
  Work inside the Bend language source repo (bendlang/bend): run and check
  .bend tests from this checkout, run the four gates (repo, test, perf, ping),
  respect the 30 s test cap and the permanent ttok token caps, know which core
  files must not be edited, and read the on-disk `#|` test format. Use when
  editing the language, compiler, base library, or tests in this repo, when a
  gate fails, when adding a test, or when asked to run the gates. For writing
  Bend *code* (syntax, laws, proofs, the CLI) in any project, see the
  cross-project `bend` skill at ~/.agents/skills/bend.
license: MIT
compatibility: this repo only; bun installed; `ttok` for the repo gate
metadata:
  author: tamara
  version: "1.0"
---

# Bend repo — working inside `bendlang/bend`

This skill is the *repository* guide: how to build, test, gate and edit this
codebase. The *language* itself (syntax, types, laws/proofs, the CLI) lives in
the **cross-project `bend` skill** (`~/.agents/skills/bend/`) — point users and
yourself there before writing `.bend` code, and keep the two from drifting by
not duplicating language content here.

## Hard rules (do not violate)
- **`bend2/bend.ts` is human-written — do not edit it** (AGENTS.md is explicit).
- **The ttok caps are permanent** (`gates/repo.ts`): `bend.ts` 48k, `comp.ts`
  66k, `main.ts` 16k, `safe.ts` 24k, `bendtt.lean` 64k, `README.md` 4k,
  `GUIDE.md` 8k. **Do not raise, lower, add or remove a cap** — only Taelin
  authorizes that, and a file over its cap is made smaller by *simplification*,
  never by moving code out. `comp.ts` sits right at its cap: after any edit,
  re-check with the repo gate.
- **The gates have a 30 s cap** (`gates/_run.ts`, `CAP = 30_000`). A run that is
  not done at 30 s is killed and FAILS. The law is: find the slow test/build/
  plumbing and make it cheap, or delete it — do **not** raise the cap, catch the
  failure, skip it, or move work out of a gate.
- **Every tracked file must match the allow list** in `gates/repo.ts`. A new
  file that is not on the list fails the repo gate. Add it to the allow list the
  same way as its siblings (it is a one-line `allow(...)`), and note the pattern
  the allow line matches (e.g. a `tests/<ns>/*.bend` glob already covers new
  tests in `tests/`).
- **This skill lives under `.agents/` and is tracked.** It is on the allow
  list in `gates/repo.ts` (`allow(/^\.agents\/skills\/[a-z0-9-]+\/SKILL\.md$/)`).
  Its `SKILL.md` has no ttok cap (like the other doc files). A new skill under
  `.agents/skills/` follows the same pattern; any other file type or a nested
  path is not allowed and would fail the repo gate.

## Running from this checkout
The gate and the tests always run the CLI straight from source:

```bash
bun bend2/main.ts <file.bend> --check-only     # check only (fast)
bun bend2/main.ts <file.bend>                  # check + run main
bun bend2/main.ts <file.bend> -o <out>         # build (extension picks target)
bun bend2/main.ts <file.bend> --verdict        # re-check with the proven kernel
```
(See the cross-project `bend` skill for the full CLI surface.)

## The gates
```bash
bun gates/_run.ts --gate        # all four, side by side, 30 s hard cap
bun gates/repo.ts --gate        # allow list + ttok caps   (PASS: n / n)
bun gates/test.ts --gate        # every test, sharded on the mini cluster
bun gates/perf.ts --gate        # benches vs the pins (perf only, slower)
bun gates/ping.ts --gate        # installer / compiled bend / release on a hub
bun gates/safe.ts --gate        # verdict vs BendTT kernel (tests | hub)
```
- A gate that passes prints one line (its verdict); a gate that fails prints its
  full output (a crash shows its message, not Bun's banner).
- `--gate` suppresses per-fail detail and prints just the verdict line.
- `gates/test.ts` shards the tests one per live mini; each runnable test is
  checked, interpreted, run in JS and run in C, each under a 5 s alarm.
- Pins live in `bench/*/_pin_/` (one file per hardware; `apple_m4` is the
  gate's, `apple_m4_max` the record). `bun gates/perf.ts --pin` writes the
  medians of three runs.

## The test format (on disk)
A test is a **single `.bend` file** in `tests/<ns>/<name>.bend`, with no test
framework: the trailing `#|` lines *are* the assertion.
```bend
# a comment; the file is a real Bend program
import Base
def main(): ...
#| True{}            <- everything from the first #| onward must equal stdout,
#| more lines        <- trimmed of trailing whitespace and leading/trailing blank
```
- The expected output is `src` filtered to `#|` lines, each stripped of its
  `#| ` prefix, joined by `\n`, then trimmed.
- A test **runs** only if it has a `def main`/`law main` *and* imports `Base`
  (a program compiles only over Base). A test without Base is checked +
  interpreted only.
- A test whose expected output starts with `SOME PROOFS FAIL` or `Error:` is a
  *negative* test (it asserts the check/run fails).
- Foreign effects in a test are pulled in with `import "./<name>.c"` /
  `import "./<name>.js"`; a foreign with no twin for a lane drops that lane.
- The 22 namespaces are the `tests/*` dirs (`parse`, `check`, `proof`, `io`,
  `run`, `compile`, `spec`, `grade`, `cost`, `halt`, `gfx`, …).
- **Adding a test:** drop the `.bend` file into the right `tests/<ns>/`; the
  allow line `tests/<ns>/*.bend` already covers it — no allow-list change.

## Editing the core files
- `bend2/bend.ts` — the language (parser, theory, checker). **Hands off.**
- `bend2/comp.ts` — the one compiler: C runtime (host + device from one
  source), the C emitter, the JS/Metal/CUDA emitters. At its 66k cap — keep
  edits size-neutral or shrinking.
- `bend2/main.ts` — the CLI and the `.bend` loader (bun plugin + node hook).
- `bend2/base.bend` — the prelude; `bend2/effs/` — per-backend IO sources
  (each effect has a `.c` and a `.js` twin; related effects may share a file).
- `bend2/safe.ts` — the `--verdict` / `.bendtt` elaborator.
- `bend2/bendtt.lean` — the proven kernel + its claims; built once with Lean
  4.34 for `--verdict`. The cross-project skill's `references/bendtt.md`
  (`~/.agents/skills/bend/`) explains its claims and the Dershowitz–Manna
  termination measure.

## Working rules
- **Confirm a fix with its own evidence:** a fix to the checker/compiler should
  come with a test in `tests/` that fails before the fix and passes after. Run
  the repo gate after any core-file edit (the cap), and the test gate to prove
  no regression.
- **Keep `comp.ts` under cap.** After editing it, run `bun gates/repo.ts --gate`
  — a 66,010-byte `comp.ts` fails. `git log` on this file shows past
  size-reclaiming commits (`78251103`).
- The site repo `../bend-lang.com` (a sibling checkout) is read by
  `gates/ping.ts` and `bend2/docs/gen_charts.ts` (or at `$SITE_REPO`); it holds
  the installers, hub and per-platform executables. Not part of this repo's gate.

## Repo map
| Path | What |
|------|------|
| `bend2/bend.ts` | the language (human-written; **do not edit**) |
| `bend2/comp.ts` | the one compiler + runtimes (C, Metal, CUDA, JS) |
| `bend2/main.ts` | the CLI and the `.bend` loader |
| `bend2/base.bend` | the prelude |
| `bend2/effs/` | IO effect sources, `.c` + `.js` twins |
| `bend2/safe.ts`, `bend2/bendtt.lean` | `--verdict` elaborator + proven kernel |
| `bench/runtime/`, `bench/checker/` | benches + their twins and `_pin_/` |
| `tests/<ns>/` | the tests (the `#|` format above) |
| `gates/` | the four gates + `_run.ts` (30 s cap) |
| `demos/` | one dir per demo, each with its `LAWS.bend`/`PROOF.bend` |
| `guide/`, `paper/`, `media/` | the guide, the papers, the film and charts |
| `evals/` | the models' proof arena — **not counted** by the repo gate |
| `tools/bend-fmt-lsp/`, `tools/bend-lint/` | the in-repo formatter LSP + linter |
