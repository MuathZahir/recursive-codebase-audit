# Issue Taxonomy — with concrete examples

This is the catalog agents hunt against. Tell agents to look for **any** issue — this is
a floor, not a ceiling. Each class below has tells (what to grep/look for) and an example.

Two rules bind the whole catalog:
- **Skip anything tooling already enforces** (linter, compiler, formatter). Report what tools
  *can't* see — cross-feature duplication, inconsistency, depth.
- **The repo overrides the catalog.** A documented standard (`CODING_STANDARDS.md`) or a decision
  in `docs/adr/` wins: a deliberate choice is not a finding. Name things in `CONTEXT.md`
  vocabulary so findings from different agents merge.

## 1. Dead code
**Tells:** exported symbols with zero importers; functions only referenced by their own
file or tests; orphaned files; feature flags that are always one value; commented-out
blocks; `if (false)` / unreachable branches; abandoned use-cases left after a migration.
**Verify before claiming:** search for real importers across the whole repo, not one
directory. The single most common failure of this whole audit is over-claimed dead code —
a symbol that *looks* unused but is wired through a barrel export, a DI/composition root,
a string-keyed registry, or another feature.
**Example:** `resolveDirectModel` exported from `direct-model.ts` but referenced only
inside its own file → dead. But `composeStockMontageVideo` *looks* dead in its own
feature yet the `export` feature imports it → NOT dead.

## 2. Redundant / repeated logic
**Tells:** the same 10-30 line block copy-pasted across sibling features; per-feature
`retry.service.ts` / `progress.service.ts` that are byte-for-byte wrappers around a shared
helper; the same S3-upload / status-mapping / error-classification implemented N times.
**Report it as:** "X is duplicated in A, B, C → extract shared module Y." Name the
suggested shared module so a parent agent can confirm it against other reports.
**Example:** five job handlers each re-implement validate→status→progress→notify→refund
because the shared factory only abstracts "how to run," not "what to run."

## 3. Inconsistency across similar features
**Tells:** parallel features (several generation types, several CRUD resources, several
pipeline stages) that do the same job with different structure, naming, error handling, or
timing — for no principled reason. The tree is uniquely good at finding these because no
single feature can see its siblings.
**Example:** four features charge cost at enqueue via a shared callback; a fifth charges
inline at a different moment with a bespoke path — same primitives, divergent seam.

## 4. Wrong tool for the job
**Tells:**
- Manual object/shape validation (`if (typeof x.foo !== 'string') throw…`, hand-walked
  property checks) where **zod** / the project's schema lib already exists. Look for
  `JSON.parse(...)` not immediately validated by a schema.
- Hand-rolled HTTP (`fetch` + manual `AbortController`/timeout/retry) where a shared http
  client / `with-retry` helper exists.
- Bespoke date arithmetic instead of the date library; string concatenation for SQL
  instead of the query builder/ORM; manual deep-clone/equality instead of the utility.
- Re-implementing a framework feature (routing, DI, caching) by hand.
**Example:** a service hand-parses an LLM JSON response field-by-field and throws custom
errors, while every sibling uses `generateStructured(schema)` with a zod schema.

## 5. Weak / unsafe typing
**Tells:** `any`, `as any`, `as unknown as T`, `@ts-ignore` / `@ts-expect-error` without a
reason, `!` non-null assertions on values that can be null, function parameters/returns
left untyped, exported APIs with inferred-`any` boundaries, `object` / `Function` types,
stringly-typed enums (`status: string` instead of a union), `Record<string, any>` payloads.
**Why it matters:** these are the points where the type system stops protecting refactors —
exactly the code most likely to break silently during the cleanup the audit is enabling.
**Example:** `as any` on a money/credits value crossing a cost seam — the one place you
least want the compiler to look away.

## 6. Shallow modules
**Tells:** a wrapper whose body is one delegating call; a "service" that only forwards to a
repo; indirection layers that exist "for testability" but add no behaviour; understanding one
concept makes you bounce between many tiny modules (no locality). Apply the **deletion test**:
imagine inlining it at every call site. If nothing is lost, it was a pass-through (collapse it).
If real complexity now reappears at every caller, it was deep (keep it). Remember the interface
is the test surface — a deep module behind a small interface is a *smaller* thing to test.
**Example:** `podcast-queue.ts` whose entire body is `return enqueue(QUEUE.PODCAST, job)`
→ inline it. Contrast a retry service that, if inlined, would scatter backoff math across
20 callers → that one earns its keep (and should be the *single* shared one).

## 7. Deepening / extraction opportunities
**Tells:** the inverse of #2/#3 — you can see the shared module that *should* exist. Propose
it in locality/leverage terms: what complexity it concentrates, what leverage callers gain,
rough effort and blast radius. Rank against other opportunities.
**Guard against premature abstraction:** **one caller is a hypothetical seam; two callers is a
real seam.** Don't propose extracting a shared module until a second caller actually exists —
the same discipline that keeps you from over-claiming dead code keeps you from over-claiming
"should be one module."
**Example:** "Fold validate/status/progress/notify/refund into the worker factory →
~200 LOC deleted, refund-on-failure becomes structural instead of a per-catch ritual."

## 7b. Structural smells across files (the tree's specialty)
These only become visible when reports from sibling features meet at a parent — exactly what
the recursive tree is for. Each is a judgement call, not a hard violation.
- **Shotgun surgery** — one logical change forces edits scattered across many files.
  *Tell:* "to change retry I had to touch 5 files." → gather what changes together into one module.
- **Divergent change** — one module edited for several unrelated reasons. *Tell:* a file that
  shows up in every feature's diff for different causes. → split so each module changes for one reason.
- **Repeated switches** — the same `switch`/`if`-cascade on the same type recurs across features.
  → replace with polymorphism, or one map both sites share.
- **Data clumps / primitive obsession** — the same few fields keep travelling together as loose
  params, or a primitive/string stands in for a domain concept (`status: string`, a raw
  `userId: string` everywhere). → bundle into one type, named in `CONTEXT.md` vocabulary.
- **Speculative generality** — abstraction, params, or hooks added for needs nothing actually
  has. *Tell:* a config flag with one value, an interface with one implementer, a `<T>` never
  instantiated twice. → inline it back until a real (second) need shows.

## 8. Removal blast radius (only when deleting something)
Two distinct buckets, and conflating them is dangerous:
- **Only-reachable-from-doomed-code** → delete it too. Trace every thread: shared helpers,
  db tables/columns, enum/union members (and every `switch`/map that must lose a case),
  config, queue/worker registrations, contracts, frontend components, routes, i18n keys,
  cost/pricing rows.
- **Entangled shared code** (used by the doomed thing AND survivors) → separate surgically,
  never blind-delete. Note the **deletion order** if removing one thing requires unwiring
  another first.
- **Recompute, don't just delete** aggregates that *summed* the doomed thing (e.g. a total
  cost that added the deleted items' costs).
