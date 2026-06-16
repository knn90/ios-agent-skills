---
name: ios-skill-consolidate
description: "Repo-maintenance skill (NOT shipped to consuming projects). (Re)generates a consolidated specialist skill (e.g. ios-swiftui-expert, ios-concurrency-expert) from its SOURCES.yaml — audits existing sources for staleness, discovers newer/better sources on the web, then fetches, synthesizes, and writes the consolidated SKILL.md. Run with no target to pick from a menu of available skills (or `all`). Use to create or refresh a specialist skill and keep it from going stale."
argument-hint: "[target-skill | all] [--discover | --no-discover] [--open-pr]"
---

# Skill Consolidate — build/refresh a specialist skill from curated + discovered sources

Maintenance tool for THIS repo. Keeps a consolidated specialist skill (SwiftUI, concurrency, …)
current by re-synthesizing it from pinned upstream sources **plus** a freshness audit that finds
new sources and flags dead ones. Output is one consumable `SKILL.md` (+ `references/`).

> **Not a consuming-project skill.** It edits skill *content* in this repo — don't ship it into
> an app's `.claude/skills/`.

## Inputs
- `target-skill` — the specialist to (re)build, e.g. `ios-swiftui-expert`. Its folder must hold a `SOURCES.yaml`. **If omitted, a selection menu is shown (Step 0).** Pass `all` to consolidate every specialist in turn.
- `--discover` — run Step 1 (Source Audit & Discovery). Default **on**; the freshness guard.
- `--no-discover` — skip discovery; just re-pull the current sources (fast refresh).
- `--open-pr` — finish by opening a PR with the changes instead of leaving them in the tree.

## Process

> **Runs per selected target.** Step 0 picks the target(s); Steps 1–5 then run **once per target**
> (for `All`, loop over every specialist).

### Step 0 — Select target(s)
- If a `target-skill` arg was given, use it. If `all` was given, select every specialist.
- **Otherwise, present a menu.** Discover the available specialists by scanning for folders that
  contain a `SOURCES.yaml` (the specialist skills live under `ios-specialists/`). Ask via
  `AskUserQuestion` with one numbered option **per skill found**, plus an **"All"** option:
  ```
  Which skill do you want to consolidate?
    1. ios-swiftui-expert
    2. ios-concurrency-expert
    3. …                       (one per ios-specialists/*/ that has a SOURCES.yaml)
    n. All
  ```
  The list is **built dynamically** from what's on disk — dropping in a new
  `ios-specialists/<name>/` with a `SOURCES.yaml` makes it appear automatically, no edit here.

### Step 0.1 — Load (for the selected target)
Read `{target}/SKILL.md` + `{target}/SOURCES.yaml` (sources, `domain`, `discovery` config, licenses).

### Step 1 — Source Audit & Discovery  ← the freshness guard (skip with `--no-discover`)

**1a. Staleness sweep — kill dead/outdated sources.** For each `status: active` source:
archived? no commits in > 12 months? README says "deprecated / unmaintained"? guidance uses
superseded APIs (e.g. pre-`@Observable`)? → propose `status: retired` **with a reason**.

**1b. Discovery — find newer/better.** Bounded web research:
- `WebSearch` the domain using the `discovery.queries` seeds + the current year, AND check the
  `authority_anchors` (Apple docs / WWDC / HIG) for guidance changes.
- Collect candidate sources **not already listed**.

**1c. Vet candidates** against explicit criteria — drop anything that fails:
| Criterion | Test |
|---|---|
| Maintained | commits within ~12 months / not archived |
| Authoritative | credible author, strong adoption (stars/refs), or official |
| Relevant | squarely within `domain` |
| License-compatible | permits derivation + redistribution (else cite-only) |
| Non-duplicate | adds something the active set lacks |
| Actually good | a quick read confirms quality, not SEO filler |

Rank survivors; keep the top few.

**1d. Propose — never auto-add.** Present **sources to retire** (with reason) + **candidates to
add** (with why + license). **Human approves.** Apply approved changes to `SOURCES.yaml`
(`status`, reasons). Discovery never silently changes the source set.

### Step 2 — Fetch
For each `status: active` source: fetch at HEAD (`gh` / clone / `WebFetch`); record `pinned_commit`
+ `last_synced`. Confirm `license` (fill if `TBD`); anything non-redistributable → **cite/link only**.

### Step 3 — Extract (fan-out)
One sub-agent per source → pull its domain guidance as structured notes: *claim · rationale · code idiom · source*.

### Step 4 — Synthesize
Merge into `{target}/SKILL.md` (overflow detail → `references/`) under the **house style**:
- **Precedence — the `primary: true` source wins.** The official source (Apple / Swift.org docs,
  WWDC, HIG — whichever the skill's `SOURCES.yaml` marks `primary: true`) is the **source of truth**
  and overrides community sources on any conflict, gated to the project's `min_ios`. Community
  sources fill in practical patterns and mistake heuristics, not semantics.
- **Dedupe** overlapping advice; **flag** genuine conflicts in a "Contested" note (don't silently pick).
- **Organize by topic** (state, composition, layout, performance, navigation, animation, accessibility, testing).
- **Attribute** each non-obvious recommendation to its source.

### Step 5 — Record + report
Update `SOURCES.yaml` (`last_consolidated`, SHAs, dates, status changes). Emit:
- **Source-audit summary** — retired / added / unchanged (with reasons).
- **Content diff summary** — what changed in the guidance + any contested points.

Then **stop for human review**. With `--open-pr`, open a PR; else leave it in the working tree.

## Modes
- **On-demand:** `ios-skill-consolidate ios-swiftui-expert` (add `--no-discover` for a quick same-sources refresh).
- **Scheduled:** a monthly routine running `--discover --open-pr` so freshness checks surface as PRs you review — keeps skills from rotting without daily babysitting.

## Cost control
"Quick research," not a thesis: cap discovery at a handful of targeted searches + light fetches.
Full discovery on the scheduled pass; on-demand refreshes can `--no-discover`.

## Constraints
- **Official docs are the source of truth** — the `primary: true` source (Apple / Swift.org) always wins on conflict; community sources never override it.
- **Human-gates the source set** — discovery proposes, you approve adds/retires.
- **Licensing first** — verify each source's license before copying; non-permissive → cite, don't vendor.
- **Reproducible** — pin commits so a re-run is deterministic and upstream changes are diffable.
- **Never auto-commit / auto-push** — output for review.
- **Generalizes** — same flow rebuilds any specialist (`ios-concurrency-expert`, `ios-testing-expert`, …); each has its own `SOURCES.yaml`.
