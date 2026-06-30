# Agent Output Structure

Every agent in the tree returns this exact section layout (and writes the full version to
the file path its parent assigns, then returns it as the final message). Uniform structure
is what lets a parent merge child outputs and spot the same problem named twice — so do not
improvise a different shape.

Cite `file:line` (or `file`) for every claim. Trace **full flows** — entry point → use-case
→ service → shared module → storage → response — naming files at each hop, so the agent
above can see where your flow overlaps a sibling's. Mark uncertainty explicitly.

## Per-finding fields (every finding carries these)

So the parent can merge, vet, and rank mechanically:
- **Canonical key** — `<smell>:<module>`, e.g. `redundant:retry`, `inconsistent:cost-charging`.
  This is how the parent groups the same finding from two agents — pick the key carefully.
- **Kind** — `violation` (objective: verified dead code, `as any` on a money seam) or
  `judgement call` (heuristic: shallow module, "should be one module"). Distinguishes a clean
  work item from one that needs a decision first.
- **Evidence** — `file:line` **plus a short quoted hunk**, so the parent vets without re-finding it.
- **Confidence** — always. **Effort** (S/M/L) and **risk** matter most on *judgement calls and
  extraction opportunities*, where the parent's value × confidence × (1/effort) ranking turns on
  them; a verified dead export can stay terse (effort S, risk low) rather than carry the full set.

## The section layout

A fixed report layout — it *groups* the taxonomy's issue classes into report sections, so these
**lettered sections** don't map one-to-one onto the numbered taxonomy. Fill every section that
applies and keep the letters so the parent can merge mechanically.

**A. Flow map** — the end-to-end flow(s) for this scope, with file paths at each hop. A
mermaid diagram or a tight indented list. This is the backbone the parent uses to align scopes.

**B. Dead code** — each item: `file:line` — what it is — why it's dead — how you verified
(importer search) — confidence. (Remember: verify before claiming.)

**C. Redundancy & repeated logic** — what is duplicated, every place it appears, and the
single shared module it suggests (name it). This is what the parent diffs against other agents.

**D. Inconsistencies & structural smells vs sibling features** — where this scope does the
"same" job differently from its siblings, and the specific difference (concrete, not vibes).
Also flag *shotgun surgery* (one change forced edits across many files), *divergent change*
(this module is edited for many unrelated reasons), *repeated switches* on the same type, and
*data clumps / primitive obsession* — these are what the parent confirms across agents.

**E. Wrong-tool & weak-typing findings** — manual validation that should be zod/schema;
hand-rolled HTTP/retry/date/SQL; `any`, unsafe casts, `@ts-ignore`, untyped boundaries.
`file:line` each.

**F. Shallow modules / deletion-test candidates** — pass-throughs and indirection that add no
leverage, with the deletion-test reasoning.

**G. Deepening / consolidation / extraction opportunities** — ranked. Each: the module to
extract, rationale in locality/leverage terms, rough effort and blast radius. Only propose a
shared module once a **second** caller exists (two adapters = real seam); flag single-caller
ideas as speculative.

**H. Cross-cutting candidates to pass UP**  ← the most important section for the parent —
shared logic you saw that probably ALSO exists in sibling features. Name the suspected shared
module precisely (e.g. "retry," "progress emission," "cost charge/refund," "status mapping,"
"upload helper," "validation schema") **and tag it with its canonical key** so the parent can
match it across agents by key and turn N local smells into one repo-wide finding.

**I. (If removing something) Blast radius in my scope** —
- Files/symbols ONLY reachable from the doomed code → delete.
- ENTANGLED shared code (doomed + survivors) → surgical separation; note any deletion order.
- db / enum / config / cost / contract / i18n / route references to remove.

## Consolidation note for orchestrators

Before returning, an orchestrator must do more than staple its leaves together:
- Merge duplicate findings; when leaves disagree (especially on dead code), verify against
  source and report the resolved answer, not both.
- Promote any smell named by ≥2 leaves into a section-H cross-cutting candidate.
- Rank section G across all leaves, not within each.
