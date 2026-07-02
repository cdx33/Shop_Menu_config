# Venn Shop Menu Generator

Turns a menu design (website URL, screenshot(s), Figma export) + a `storekey` into
paste-ready `MenuItem[]` JSON for the Venn Store Editor document
**"Shop Menus - Dashboard only"**.

## Source of truth
`docs/spec.md` is authoritative. Read it before any structural change. If this file and
the spec ever conflict, the spec wins.

## Non-negotiable invariants (Store Editor validation rejects violations)
1. Every node has all four keys: `image`, `title`, `link`, `children`. Always.
2. This generator emits ONLY `link.type` values `"category" | "page" | "empty"`.
   Other types (`tabBarId`, `product`, `nativeWebPage`, `brands`) exist in legacy data —
   we never emit them.
3. Category links: `payload` = the **numeric collection ID as a string** (`"159983206452"`),
   never a handle. `readableTitle` = the exact collection name, always set.
4. Container tab (level-1 grouping with no destination): `{ "type": "category", "payload": "" }`.
5. Section header (grouping row inside a tab): `{ "type": "page", "payload": "", "readableTitle": "" }`.
6. Max depth = 3. Level-3 `children` MUST be `[]`. Deeper input → auto-flatten
   (promote level-4 leaves up one level) + one warning per promoted node.
7. Unresolved label: `payload: ""`, `readableTitle: "<label as searched>"`, warning emitted.
   NEVER guess or invent a collection ID. An empty payload is correct behaviour; a wrong ID is a bug.
8. `image` is always `""` (runtime backfills from the collection image) unless a real URL is supplied.
9. Duplicating tabs (e.g. MEN → WOMEN): re-resolve every leaf against the target tab's
   collections. Never reuse IDs across tabs.
10. Output: pretty-printed JSON, 2-space indent. Default deliverable = the `menu` array only.
    Full-document variant (adds `timestamp`, optional `storekey`) only on request.

## Architecture rules
- `packages/core` is pure and deterministic: no network, no LLM, no side effects, no `Date.now()`
  except in `toFullDocumentJson`. The category list is injected as data.
- Vision/LLM extraction happens ONLY at the extraction edge: the Cursor agent's own
  vision turns images into an `ExtractedMenuNode` tree (`tmp/tree.json`); `adapters/url`
  handles websites deterministically. Resolution, matching, validation and emission are
  plain code. Never let a model write an ID.
- Matching is scored, in priority order:
  exact name (1.0) → exact handle (0.95) → gender-prefixed name, hint from ancestor tab title,
  e.g. "HOODIES" under MEN → "Men's Hoodies" (0.85) → contains (0.7) → token overlap (≤0.6).
  Status thresholds: ≥0.9 `auto`; 0.6–0.89 `guessed` (human must confirm); <0.6 `unresolved`.
  Two candidates within 0.05 of each other → `ambiguous`, list both.
- Every pipeline stage returns `{ data, warnings }`. Warnings are first-class output,
  carried through to the end and always shown to the user.

## Definition of done (applies to every task)
- `npm test` green, including TC1–TC5 from spec §13 as named tests.
- `validate` mirrors the Joi schema in spec §5 exactly, plus one extra guard:
  `type === "category"` with a non-empty payload must match `/^\d+$/`
  (catches the #1 anti-pattern: a handle in `payload`).
- The real Couture Club example (spec §6) passes `validate` unchanged — use it as a fixture.
- No `any` in core's exported types.

## Commands
- `npm test` — full suite
- `npm run cli -- <command>` — pipeline CLI (see cli/README)
