<p align="center">
  <img src="assets/hero.png" alt="Recursive Codebase Audit — a ROOT agent branching into orchestrators, each branching into leaf agents" width="100%">
</p>

<h1 align="center">Recursive Codebase Audit</h1>

<p align="center">
  A <a href="https://claude.com/claude-code">Claude Code</a> skill that audits a large codebase with a
  <b>recursive tree of parallel sub-agents</b> — surfacing the problems a single-pass review can't.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Claude%20Code-skill-d9a441?style=flat-square" alt="Claude Code skill">
  <img src="https://img.shields.io/badge/mode-analysis%20only-3a3a3a?style=flat-square" alt="analysis only">
  <img src="https://img.shields.io/badge/install-npx%20skills%20add-d9a441?style=flat-square" alt="install via npx skills add">
  <img src="https://img.shields.io/badge/license-MIT-3a3a3a?style=flat-square" alt="MIT license">
</p>

## Why it works

A single agent runs out of context long before it has seen a large codebase. So instead of one
reviewer, this skill builds a **tree**: a root maps the repo and fans out orchestrators, each
spawns leaf agents, and **findings bubble upward**.

That upward flow is the whole point. *"Podcast hand-rolls retry"* is invisible from inside podcast
— it only becomes a finding when the same smell meets its siblings at a parent:

<p align="center">
  <img src="assets/bubble-up.png" alt="Leaf findings flow upward: three leaves each spot a local retry smell, an orchestrator sees two of them differ, the root concludes five features hand-roll retry and should share one module" width="82%">
</p>

**The same problem repeated across features is invisible from inside any one feature.** That's why
the tree beats N independent reviews.

## The five-step flow

<p align="center">
  <img src="assets/five-step-flow.png" alt="Five steps: Map, Briefing, Fan out, Consolidate, Deliver" width="100%">
</p>

The root does the cheap **map**, then writes one **shared briefing** so every agent hunts the same
classes and returns the same shape. Agents **fan out** in parallel; the root **consolidates** — vet
each finding against source, group by canonical key, and the cross-feature patterns are the prize —
then **delivers** one ranked report.

## What it hunts

| | |
|---|---|
| **Dead code** | exports with no importers, unreachable branches, orphaned files — always importer-verified before claiming dead |
| **Redundant logic** | the same thing implemented per-feature that should be one module |
| **Inconsistency** | parallel features doing the same job differently for no reason — the tree's specialty |
| **Structural smells** | shotgun surgery, divergent change, repeated switches, data clumps, speculative generality |
| **Wrong tool** | hand-rolled validation instead of zod, bespoke HTTP/retry, raw SQL, `JSON.parse` with no schema |
| **Weak typing** | `any`, unchecked casts, `@ts-ignore`, untyped boundaries |
| **Shallow modules** | interface ≈ implementation — plus the *deepening* opportunities that would earn their keep |
| **Removal blast radius** | what's *only* reachable from doomed code (delete) vs *entangled* with survivors (separate) |

A few principles keep it honest: **skip what the linter already enforces**; a **documented decision
(ADR / `CONTEXT.md`) overrides** the taxonomy; **sub-agents over-report, so every finding is vetted
against source**; and **one adapter is a hypothetical seam, two is a real seam** — no shared-module
proposal until the second caller exists.

## Scale & cost

Analysis only — it **finds and ranks, it never edits code.** The deliverable is a ranked diagnostic
you hand off to whatever turns findings into work (issues, tasks). It's also a deliberate token
spend that scales with the tree:

<p align="center">
  <img src="assets/fan-out.png" alt="Example fan-out: 1 root times 6 orchestrators times 3 leaves is about 25 agents in parallel, one shared briefing, findings merged by canonical key" width="88%">
</p>

If the scope fits in one agent's head, a normal review is the better tool — this is for **breadth**.

## Install

```bash
npx skills add MuathZahir/recursive-codebase-audit
```

Claude Code discovers it on the next session.

## Usage

Reach for it when the codebase is large and genuinely multi-feature, and a repo-wide pattern is the
real question:

> review the whole codebase for dead code and duplication
>
> why is every generation type slightly different?
>
> plan the removal of feature X and everything only it uses

## What's inside

```
SKILL.md                          the skill — when to use it and the five-step flow
references/
  briefing-template.md            the shared briefing handed to every agent
  issue-taxonomy.md               the full issue catalog with examples
  agent-output-structure.md       the report layout agents return (so outputs merge)
```

## License

MIT · Diagram art generated with [Nano Banana 2](https://inference.sh) (Gemini 3.1 Flash Image).
