---
name: empirical-verification
description: "Use when proving a feature is real, not just a flag flip."
version: 1.0.0
author: hermes-autocurated
---

# Empirical Verification — prove the claim, not the switch

A first-pass deliverable often proves the *wrong thing*: a flag flipped, a function
exists, a build passed. None of those prove the feature *behaves*. When the user
challenges with "is this the best you can do?" or "did you actually verify X?",
the answer is a second, empirical pass that asserts the real side effect.

## The core failure mode
- **Boolean-flipped ≠ behavior-rendered.** A test setting `window.FEATURE_X = true`
  and asserting `FEATURE_FLAGS.FEATURE_X === true` only verifies switch wiring.
  It says nothing about the gated code executing. Worse: if the gated path has an
  unrelated early-return (e.g. `if (!game.isOnline) return` in a draw routine),
  booting the wrong mode makes the test pass while the feature silently never runs.
- **Function-name-exists ≠ feature-wired.** A matching exported symbol with zero call sites
  in the consuming component is dead code. Grep the ACTUAL call sites.
- **Wired into the WRONG copy of the handler.** A refactor can leave an OLD handler
  file exporting the same plausible symbol (`drawLineHandler`) that still typechecks
  and unit-tests green in isolation, while the client calls a DIFFERENT module. The
  rule lands in the dead copy; tests written against the dead copy pass; production
  never runs the rule. Trace the client's actual import to the live handler and place
  the rule there. Full recipe: the `convex-operations` skill, reference
  "live-path-wiring-and-deploy-gate".
- **Build/test green ≠ claim true.** Green tests can still assert the wrong thing — and a
test that is never COLLECTED proves nothing at all while the suite still reports green
(see "A test that never runs" below).

## The rule
Before claiming a feature "works" / "ships" / "renders", assert a VISIBLE SIDE
EFFECT of the gated behavior itself, driven through the real code path:
1. Boot/invoke the correct mode (the one the guard actually allows).
2. Call the real function (not a stub).
3. Assert an output: a canvas op ran, a DOM node appeared, a state field changed
   to the expected value — not merely that a flag boolean equals true.

## Split responsibilities (so each test is honest)
- **E2E** proves the *bridge* (e.g. `window.FEATURE_*` → `FEATURE_FLAGS` mirror).
  Do not also re-assert drawing in E2E — it needs internal modules exposed on
  `window`, which is a fragile/over-reaching change.
- **Unit / integration** proves the *behavior* — that the real draw/method
  executes and produces output. See `references/render-proof-canvas-mock.md` for
  the jsdom canvas-mock technique (no `canvas` pkg, no `CanvasRenderingContext2D`).

## Second-pass discipline (from "is this the best you can do?")
1. Re-derive every claim: what TOOL proves it? Run the probe.
2. Diff pass-1 vs pass-2 as a confession table (said X → verified Y).
3. Apply corrections and re-verify mechanically.
4. Codify the new scar into a skill.

## Verify framework/library behavior before asserting it

A review that claims *"Convex `db.patch` validates the patch object against the schema, so this throws"*, *"React batches all state updates in event handlers"*, or *"Postgres serializes concurrent writes"* is asserting **framework behavior**. If the claim is load-bearing for a finding (e.g. "this will throw at runtime"), verify it against the framework's own types, docs, or source before stating it as fact. The verification is usually one read away — `node_modules/<pkg>/dist/*.d.ts` for TypeScript libraries, the framework's own docs site, or a minimal probe (`npx tsx -e '...'`).

**The shape of the fix is evidence about the framework's semantics.** When a developer's shipped fix only makes sense under a specific framework behavior, that behavior is likely correct — don't assume the fix is broken and the developer forgot a step. Example: a Convex handler that patches `{ newField: true }` without adding `newField` to the schema *only works* if the server allows extra fields. If you see that pattern in shipped code, the framework probably allows extra fields — read the types to confirm before flagging the code as broken. The same logic applies to: a React component that mutates state in a render and ships fine (probably batched under strict mode / a concurrent renderer), a SQL migration that drops a column without backfilling (probably the ORM handles it), a Node API that uses `Buffer` in a file without `"use node"` (probably a Web Streams–compatible shim).

**Caveat (2026-09-03, ShapeKeeper v2 review): the shipped-fix heuristic FAILS when the
shipped fix is itself based on a misunderstanding of the framework.** The ShapeKeeper v2
fix added `await ctx.db.patch(square._id, { multiplierRevealed: true })` without
declaring `multiplierRevealed` in the schema. Applying the heuristic literally:
"the fix only works if the server allows extra fields, so the server probably allows
extra fields, so the code is fine." This was codified as a pitfall in
`convex-backend-patterns` §6. It was wrong. The Convex `db.patch` docstring says
"New fields are added" (which describes the merge step), but the schemas page
(`https://docs.convex.dev/database/schemas`) says "schema validation enforces your
schema on every `db.insert`, `db.replace`, and `db.patch` call." The merge runs
first, then validation throws. The fix is broken at runtime; the heuristic was
fooled by a plausible-looking code shape that was itself based on a wrong reading
of the docstring.

**The corrected heuristic:** before applying the "shipped fix implies framework
behavior" rule, verify the framework behavior independently — read the
**validation / error-handling docs**, not just the operation docstring. A patch
docstring describes what the merge does. A schemas/validation page describes what
the server rejects. Both are needed; the first alone produces confident wrong
answers. Cost of the verification: one curl or one `grep` in `node_modules`. See
`convex-backend-patterns` §6 "Convex `db.patch` rejects fields not in the schema"
for the full corrected lesson with source URLs.

**Diagnostic question when a claim contradicts shipped code:** *"Why does this code work in production if my claim is true?"* If the only answer is "the developer never tested it," verify with a probe before stating the claim. If the answer is "the framework actually behaves differently than I thought," update the claim and note the framework's actual behavior.

**The verification cost is one file read or one probe.** A 30-second `grep` in `node_modules` or a 5-line test script is cheaper than a 500-line review that the user has to correct. See [Convex schema-light drift](#convex-schema-light-drift) below for the worked example.

## A test that never runs still reports green

A green total is not evidence that a specific test executed. Runners collect only files
matching their configured globals, so a spec placed beside its source in an uncovered
directory is **silently not collected**: it never runs, never appears in the per-file
list, and cannot fail. The suite passes and proves nothing about the code it names.

- Put module/engine tests where the runner's globs actually reach (commonly `src/**`),
or widen the globs. Importing the module by relative path from a covered directory works
fine and is usually the smaller change.
- After adding a test, confirm the runner LISTS ITS FILE — read the per-file output, not
the summary total.
- **Treat a wrong DELTA as a signal.** If you add two tests and the total rises by three,
something was already uncollected, duplicated, or skipped. Find it before proceeding.
The mirror case: a count that stays FLAT after you delete a test means that file was
never being collected.

This is the strongest form of "tests pass ≠ claim true" — not a weak assertion, but no
execution at all, and it is completely invisible in the pass count.

## A skipped stage reports the PREVIOUS run's result

The sibling failure to "a test that never runs still reports green": a stage you
skipped can still print a real-looking number read from an artifact the *earlier*
run wrote. Verified 2026-10-10 — a `--skip-live` flag skipped the live model
probes yet the summary still printed "live: 3/3 slots responded", parsed from the
previous run's `verification.json`. The number was true, just not about this run.

- **Delete, don't overwrite.** Clear every generated artifact at the start of a
  run, including the ones belonging to skippable stages. An artifact from a prior
  run is indistinguishable from a current one once it is on disk.
- **A skipped stage must report `NOT TESTED`**, never a stale count and never a
  silent blank. The absence of a stage is a finding, not an empty field.
- Any report that aggregates across stages has to say *which stages ran this
  time*; "3/3 passed" without that qualifier is a claim about a pipeline you did
  not execute.

## Anti-patterns
- "Flag is true, so the feature is live." (proves wiring only)
- "The function is exported, so it's implemented." (zero call sites = dead)
- "Tests pass." (they may assert the wrong thing — or never have been collected at all)
- E2E that boots the wrong mode and asserts a boolean — a green test that proves nothing.
- **"File shrank, so the compression was a win" — when the file is structured (§-delimited entries, JSONL rows, CSV, header+body).** A `wc -c` check proves only that the on-disk size changed. The on-disk file can shrink while the structure silently breaks: 3 §-separated entries fused into one with `\n` instead of `§` between them, a JSONL row truncated mid-line, a CSV row that lost its trailing comma but kept its line break. Real incident (2026-09-02, MCM v1 memory-compaction): reported "6.14% saved, looks fine" and the size had indeed dropped — but the file had merged 3 entries into 1, and the model could no longer `memory(action=remove)` them individually. Protocol: (1) count structure markers BEFORE the edit (`text.count('§')`, `len(text.split('\n'))`, `sum(1 for _ in open(p))`); (2) run the edit; (3) count the same markers AFTER; (4) assert equality or an explicitly-expected delta. Applies to any place where the structure is the data — JSONL logs, CSV cells, package.json `scripts` keys, .gitignore rule counts, multi-line records. A "no change in count" is fine; an "I expected N-1" needs a code-level reason.

## When the user asserts a third-party version / release fact
User-supplied claims about external systems ("X is at version Y", "release Z is
out", "you're 7 updates behind") look authoritative — they often arrive richly
formatted with dates, subcommand names, and changelog bullets. They can also be
flat wrong (Reddit summary, stale blog post, mixed-up product line). Never
install/upgrade/migrate based on the user-supplied claim alone.

Verify against an **upstream authoritative source** before any side effect:

1. **The artifact's own self-report.** A CLI's `--version` and self-update
   subcommand (`agy update`, `npm outdated`, `pip index versions`, `gh release
   list`) is the fastest corroborating signal. If the binary says "you are on
   the latest", treat that as a strong vote — not gospel, but a data point.
2. **The vendor's canonical source.** GitHub Releases API
   (`GET /repos/{owner}/{repo}/releases/latest` → `tag_name`) is authoritative
   for an open-source release line. The vendor's official blog / changelog
   resolves version-line confusions (e.g. Antigravity 2.0 was a *desktop app*,
   not a CLI release — the CLI stayed on its own 1.x line).
3. **The artifact's bundled changelog.** `agy changelog`, `npm info <pkg>`,
   `cargo search`, etc. — the binary only knows what it was built with, so a
   1.x CLI's changelog will not list 2.x entries even if 2.x exists for a
   sibling product. Use this to disambiguate "is the user's X really the
   binary's X?"
4. **Only then act.** If 2/3 sources agree with the user's claim, proceed. If
   the on-disk artifact contradicts the claim, **report the contradiction
   with citations** (binary version + GitHub tag + blog quote) and ask the
   user to confirm before any destructive change. Do not silently fall back
   to a different product line.

Hard rule: never fabricate an upgrade to match a user-asserted target version.
Either the version exists upstream (verify, then install) or it does not
(report, do not install). "Describe vs deliver" cuts both ways: describing
an upgrade you did not actually perform is a worse failure than reporting
that the binary is already on the latest.

## Anti-patterns (version / release claims)
- "User said it's at 2.7.1, so I'll run the update command." (claim
  unverified; may install a non-existent version or break the working state)
- "`agy update` says I'm current, but the Reddit post says 2.7.1 exists, so
  I'll force-install from a random URL." (Reddit is not the upstream source)
- "I can't find 2.7.1 in the changelog, but maybe it's in a different
  channel — let me try anyway." (channel-hopping without evidence; back up
  the binary first if you do, and only on explicit user request)
