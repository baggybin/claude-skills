---
name: bug_hunt
description: >-
  Deep-dive bug hunt over a codebase using a swarm of read-only hunter personas
  with mandatory falsification-based verification. Use when the user asks for a
  "bug hunt", "deep dive review", "adversarial review", or "find bugs in
  <area>" — especially multi-repo workspaces, "is X ever called / wired"
  questions, or anywhere a confident but wrong headline finding would be costly.
  Phases: verify the search space mechanically, swarm narrow personas, triage,
  falsify headline claims with a different model family, report with coverage
  disclosure.
---

# Bug Hunt — verified swarm review

A procedure, not a vibe. The failure mode this skill exists to prevent: a swarm
of eager hunters produces a confident, well-written report whose headline
finding is false because nobody checked the search space behind an absence
claim. (Worked example: a "CRITICAL — zero production callers" finding that
survived an orchestrator spot-check because both hunter and checker searched 2
of 3 repos — the caller lived in the third. A *different model family* caught it
in minutes.)

Two root causes, and both must be designed against:
1. **Search-space omission** — the declared root list is wrong, so every
   hunter is confidently blind in the same way. This is a **Phase 0** defect and
   no downstream phase can catch it unless Phase 0 verifies *itself*.
2. **Correlated blind spots** — the same model family hunts AND verifies, and
   shares the omission. Fresh *context* is not fresh *priors*; only a different
   model family (or a mechanical check) de-correlates.

Research grounding: maker/checker separation and fresh-context verifiers
([adversarial code review](https://www.augmentcode.com/guides/adversarial-code-review));
narrow specialist agents over context-dumping
([diffray multi-agent review](https://diffray.ai/multi-agent-code-review));
evidence-based findings, confidence scored independently of severity,
coverage disclosure, and a verifier that shares no context with the discoverer
([OWASP APTS reporting guide](https://github.com/OWASP/APTS/blob/main/standard/8_Reporting/Implementation_Guide.md)).

## Phase 0 — Map the workspace, then VERIFY the map (NEVER skip)

The declared root list is inherited by every hunter and the verifier. If it is
wrong, the whole hunt is confidently blind. So Phase 0 is not a model judgment —
it is a **filesystem fact cross-checked against a model judgment**.

1. **Model-declare** the roots: read every `pyproject.toml` / `package.json` /
   `[tool.uv.sources]` / `go.work` / submodule; follow re-export **shims** to
   where the real code lives (a 13-line `from other_pkg import *` shim means the
   code is in another repo — find it). List roots explicitly, e.g. `app/`
   (product code), `libs/core/` (vendored or editable kernel),
   `plugins/` (dynamically loaded extensions).
2. **Mechanically enumerate** the roots independently and DIFF against step 1:
   sweep the whole workspace for *every* project manifest
   (`find <ws> -name pyproject.toml -o -name package.json -o -name go.mod …`),
   resolve `[tool.uv.sources]` / workspace globs / submodules. **Any root the
   sweep finds that the model-declared list missed is the bug — reopen step 1.**
   (In the worked example a plain filesystem sweep for manifests would have
   surfaced the third repo.)
3. **Pin revisions.** For a multi-repo workspace, record the commit SHA of each
   root — a cross-repo caller may exist only at a *deployed* combination of
   revisions, not on each repo's default branch. State the snapshot.
4. **Resolve shims to implementations, not to imports.** Wildcard re-exports,
   `__getattr__`, generated stubs, registries — trace the public symbol to the
   code that runs. An import that resolves *outside* the declared search space
   is a hard STOP that reopens step 1, not a lead to maybe chase.
5. Identify the trust/lifecycle/reachability boundaries relevant to the theme
   (who spawns processes, enforces gates, renders status, wires features in).
6. Write the **verified** search space (roots + SHAs) into every hunter prompt
   verbatim.

## Phase 1 — Spawn the hunter swarm

The persona catalog is a **coverage checklist** — a fixed floor you can audit,
not a menu to pick from by intuition (ad-hoc-only personas make a missed bug
class indistinguishable from a model failure, and inherit the orchestrator's
"failure modes I already suspect" bias). Rules:

- **Run every catalog persona the coverage section can't explicitly justify
  excluding.** A skipped persona is a written, defended decision — never a
  silent default. Surface-specific ad-hoc personas are allowed as *additions*,
  never replacements.
- On any **multi-repo** hunt, the **WIRING** persona is mandatory (it is the one
  that structurally cannot make a partial-repo claim — reachability is its job).
- Treat the catalog empirically over time: track per-persona unique findings and
  false-positive rate; merge/split/retire/add lenses based on misses.

Each hunter is a **read-only** subagent with a narrow mandate
(one concern, full depth — specialization beats breadth; context dilution is
real), the verified search-space list, concrete entry-point files (located by
the orchestrator first — don't make the hunter guess paths), and the claim rules
verbatim.

### Persona catalog

| Persona | Hunts | Best on |
|---------|-------|---------|
| WIRING | Reachability/call-graph: is this code path actually invoked in production, across ALL roots? "built but never wired", dead registration, dynamic dispatch | multi-repo, plugins/DI, "is X ever called" |
| RACE | Interleavings, TOCTOU, shutdown/startup races, double-settling | event loops, async, multi-process |
| GATE | Enforcement bypass, fail-open defaults, missing checks on new paths | authz, policy gates, scopes |
| TRUST | Credential forgery, trust escalation, token handling, spoofable identity | auth, tokens, cross-agent calls |
| TRUTH | Status/success honesty: where "ok" is shown despite partial failure | dashboards, health, stop/kill paths |
| STATE | Shared mutable state, stale reads, persistence gaps, audit loss | stores, caches, DBs, ledgers |
| LIFE | Lifecycle: leaks, orphans, zombies, missing cleanup on error paths | process spawn, connections, tasks |
| LEDGER | Accounting: quotas, budgets, caps that under/over-count or disarm | metering, rate limits, budgets |
| EDGE | Error handling: swallowed exceptions, malformed input, framing, bounds | everything, breadth-first |

### Hunter prompt template

```
You are "<PERSONA>" — a <mandate> bug hunter (read-only, thorough depth).

VERIFIED WORKSPACE ROOTS (the complete search space — search ALL of them):
<roots + commit SHAs from Phase 0>

Entry points: <files the orchestrator already located>
Context: <the invariant this persona checks>

CLAIM RULES (mandatory):
- Every finding: severity, file:line, quoted code evidence, and a CONCRETE
  failure sequence (inputs/interleaving), not a theoretical one.
- Classify each finding POSITIVE ("this line does X" — anchored to code you
  read) or NEGATIVE ("nothing calls X" / "Y doesn't exist" / "dead code").
- For NEGATIVE claims: record the exact command AND the roots you searched
  ("grep -rn X <root1> <root2> <root3> → N"). If you did NOT search every
  declared root, label it UNVERIFIED-NEGATIVE — never claim absence from a
  partial search. A static text search can only prove "not found in the
  searched corpus", never runtime absence — flag dynamic-dispatch/DI/registry/
  reflection/RPC paths you could not rule out.
- A docstring/comment/log referencing a component you can't find is a LEAD:
  follow shims and search every root before concluding it's missing.
- Mark CONFIRMED (traced end to end) vs PLAUSIBLE (needs runtime repro).
  Note existing test coverage. Emit a COVERAGE WITNESS: roots searched, files/
  symbols traced, and what you did NOT reach.
Report under 600 words, ranked by severity. Do not fix anything.
```

## Phase 2 — Triage and dedupe (orchestrator)

- Merge duplicates across personas but keep each original's evidence as an
  **immutable attachment** — dedupe by `(symbol, failure-mode, root)`, NOT by
  paraphrased prose, so a merge can't silently turn "no caller found in repos
  A/B" into "no production callers."
- Split every finding into POSITIVE vs NEGATIVE claim lists.
- Cap severity of every NEGATIVE claim at MEDIUM until Phase 3 clears it.

## Phase 3 — Falsification pass (the part that prevents the disaster)

The verifier's job is to **break** findings. It must share neither the hunter's
context nor — for headline claims — the hunter's model *family*.

1. **Every NEGATIVE claim:** re-run the absence search across **all** roots and
   assert `roots_searched == roots_declared` (Phase 0), attaching the literal
   command output. A stamp that doesn't name its search space is invalid; a
   verification that repeats the hunter's narrower search is worse than none.
2. **Cross-family for headlines (mandatory).** Every CRITICAL, and every
   "extraordinary structural" claim ("built but never wired", "zero production
   callers", "control is theater"), MUST be disproved by a **different model
   family** — the correlated-blind-spot fix. The verifier gets the claim +
   immutable evidence + the root list, NOT the hunter's reasoning, and must
   independently reconstruct the root map first. Cross-family is a
   de-correlator, not a truth oracle — keep it paired with the mechanical
   root-count check above. HIGH findings: cross-family is sampled/risk-triggered,
   not mandatory (throughput). Cheaper fallback for the rest: the verifier
   checks off each Phase-0 root explicitly, stamp fails if any is unaddressed.
3. **Acceptance rule for absence claims.** For "unwired/zero-callers" to remain
   CRITICAL the verifier must establish ALL of: (a) the implementation exists at
   the pinned revisions; (b) every production root + integration boundary was
   searched; (c) no caller OR indirect registration path exists; (d) "production"
   was distinguished from tests/examples/dead code/inactive config. **Any
   discovered caller, skipped root, or unresolved dynamic path INVALIDATES the
   claim** — it does not merely "raise doubt". A downgraded claim ships as a
   scoped LEAD/UNVERIFIED-NEGATIVE, out of the headline.
4. **DISPUTED is a real state.** Hunter says X, cross-family verifier says not-X,
   unresolved → DISPUTED, blocked from headline pending a tiebreak. "Err toward
   FLAGGED not REJECTED" (don't silently drop a poorly-described real bug) but
   DISPUTED ≠ FLAGGED — an unresolved CRITICAL is the top *question*, not the top
   priority.
5. **Confidence ≠ severity.** Severity is impact if true; confidence is evidence
   quality. Report both, independently.
6. **Exploration, not just falsification (high-risk hunts).** Phase 3 only
   checks claims that were *found*. Add a bounded independent pass — ideally a
   second family — tasked with finding a DIFFERENT failure mode, so a class the
   swarm never hypothesized isn't invisible.

## Phase 4 — Report

Write to a dated file (e.g. `docs/reviews/<area>/BUG_HUNT_<AREA>_<date>.md`).
Required structure:

- **Scope note naming the exact roots + revisions searched** — "the codebase" is
  not a scope. A reader must see what was NOT examined.
- Summary table: severity + confidence per finding (+ claim type, verifier
  family for headlines).
- Per finding: severity, confidence, file:line, quoted evidence, concrete
  failure sequence, verification record (command + search space for negatives;
  cross-family verifier for CRITICALs), suggested fix, and the pin test that
  would fail today.
- **Coverage disclosure**: which roots/files/classes were tested, partially
  tested, excluded — and why. Built from the hunters' coverage witnesses, not
  narrated from memory.
- **"What held up" section**, each clean check citing the commands + roots that
  cleared it — an empty check must read as visibly empty.
- **Coverage-sufficiency gate before any global "all clear."** Do NOT emit
  all-clear because no findings survived; emit it only when the surface was
  bounded, revisions pinned, the root-map cross-check passed, WIRING signed off,
  every persona's exclusion was justified, and the dynamic/unexamined areas are
  listed. Say what was covered and what cannot be concluded.
- Recommended fix order, by (severity × confidence) / effort — not severity
  alone.

## Anti-patterns (all observed in the wild)

- **Trusting the declared search space** without a mechanical cross-check — the
  omission that produces the confident-false headline; Phase 0 must verify itself.
- **Context-independence mistaken for prior-independence** — a fresh-context
  verifier of the same model family shares the family's blind spots (scope
  enumeration, package metadata, search habits). Cross-family for headlines.
- **Persona theater**: hunters with names but no tools, entry points, or claim
  rules — plausible-severity fan fiction.
- **Context dumping**: whole files pasted into one mega-prompt — attention
  drowns, the middle is lost.
- **Confirming verification**: re-running the claim's own grep and calling it
  "verified".
- **Same-context judging**: the hunter (or a subagent that saw its reasoning)
  verifying its own finding. Self-review is measurably lenient.
- **Severity-only ranking**: a CRITICAL at UNVERIFIED confidence is the top
  *question*, not the top priority.
- **Green-absence reports**: no "what held up" + coverage gate ⇒ the report
  proves nothing about what was examined.
- **Vacuous search receipts**: a "zero hits" grep whose pattern can't match
  anything (e.g. unescaped `|` in a basic-regex grep collapses alternation into
  one literal string). Every absence receipt must be sanity-checked for pattern
  validity — quote the exact command and re-run it.

## Calibration (optional, for a repeatable harness)

Seed one known bug (or keep a fixed known-true finding) in a far/rarely-touched
root and assert the swarm rediscovers it. If the canary in the distant repo
isn't found, Phase 0 coverage is broken *before* you trust any "all clear" — a
direct regression test for the search-space-omission failure.

## After the report

Hand findings to a fixer as a SEPARATE step (separate context, write authority)
— hunters stay read-only, fixers fix, then a fresh verifier re-checks the changed
code. Maker/checker separation applies to remediation too.
