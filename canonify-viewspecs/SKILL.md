---
name: canonify-viewspecs
description: Author Canonify ViewSpecs — the declarative way to build a screen. Compose catalog pieces (ObjectTypes + ActionTypes) into a transport-agnostic view from 11 block types (table, detail, panel, button, composite, metric, chart, file, board, calendar, queue), then declare it to an org with view_specs.declare.
version: 2.0.0
---

# Canonify ViewSpecs — building screens without code

**Start here:** [resources/viewspec-cheatsheet.md](resources/viewspec-cheatsheet.md)
— every block, field and common declare error on one page.

**A screen with anything computed is SpringML, not a block.** A grid
(resource × day), a timeline, a builder with a live total, two versions side
by side, a value derived from other values, a page driven by a picked week —
write it as a SpringML `.springml` file with the
[`canonify-springml`](../canonify-springml/SKILL.md) skill. Every block
below is available there as a leaf component (`<Table source={…} />`) with
the same config, so a block screen that grows is moved, not rewritten.

A **ViewSpec** is a screen described as data: one JSON object whose `type`
picks the block. It names catalog pieces (ObjectTypes, ActionTypes) by their
declared names — never a table, query or URL — and the web app renders it.
Every shape is **closed**: an unknown or misplaced field is refused at declare
time.

```json
{ "type": "table", "source": { "objectType": "customer" }, "columns": ["name", "email"] }
```

**The declaration is the default.** A column with no `label` reads the
property's `label`; money renders in its declared `currency` with no `format`;
an enum renders its `valueLabels` and `valueColors`; a `panel` with no `title`
reads the action's `presentation.label`; a `detail` with no `properties` reads
`display.primaryProperties`. Derived properties carry the same facets. Declare
a view field only to override. What you can bind to:

```sh
canon objects list-types; canon actions list; canon schema describe --object customer
```

## Resources

| Page | Holds |
|---|---|
| [viewspec-cheatsheet.md](resources/viewspec-cheatsheet.md) | Every block, field and grammar row on one page; common errors |
| [viewspec-block-types.md](resources/viewspec-block-types.md) | The eleven blocks, field by field, with what the renderer decides |
| [viewspec-binding-and-sources.md](resources/viewspec-binding-and-sources.md) | Sources, aggregates (buckets, window, compare, expand, measures), filters, columns, formats, `$principal`, `$context` |
| [viewspec-recipes.md](resources/viewspec-recipes.md) | Record page, list, dashboard, board/queue workflow, forms — copy and adapt |

## Localization: who owns the words on the screen?

The active locale is resolved by user preference → org default → platform
default. An `@t:<key>` value is resolved from the org's locale bundle (a key
must exist in every locale the app declares; `@@t:` escapes the sigil). The
bundle mechanics are in the `canonify-app-authoring` skill's
[bilingual-apps.md](../canonify-app-authoring/resources/bilingual-apps.md).

| Category | Owner | No declaration / fallback |
|---|---|---|
| **Fixed platform chrome** — counts, `Search`, `Filter`, empty-state frames, `Optional fields`, `Select a record`, “not set”, submit/cancel | The platform's own locale catalogs. | English chrome. A null aggregate group uses chrome; give it an explicit enum value if it is business data. |
| **Generated-from-catalog text** — an empty state naming a type, a generated form-field label | The platform owns the sentence; the author owns the noun and the field label via `display.entityKind` / `entityKindPlural` and the property `label`. | An acronym-aware English humanizer. In a non-English UI a list never uses the English type name: its header, search placeholder, empty state and create caption fall back to generic chrome (`Poster`, `Søg…`, `Opret`). |
| **Enum values** — pills, dots, board columns, filters, grouped dimensions | The author's `valueLabels` (a literal or `@t:` key per value); a StateMachine `StateMeta.label` for its states. | The humanized raw value, neutral colour. Stored values and filter/sort semantics stay raw. |
| **Action labels** — `button`, `panel` submit, row/menu actions, generated CRUD captions | ActionType `presentation.label`, `display.actions`, or the view's `label` / `submitLabel`. | A generated create/update/delete/replace caption composes a localized verb with the author's localized type name; otherwise the humanized action id. A row, card or queue verb drops the type noun (its record names it): "Edit", "Replace" (localized), "Check in". |

An explicit view label wins for that view; a literal is never translated, and
a missing `@t:` key is a locale-completeness error, not a raw key on screen.
For a localized canon declare type names, property `label`s, every visible
`valueLabels` entry and labels for surfaced custom actions; keep stored values
and action ids stable.

**Dates and numbers: the org's clock, the reader's locale.** Timestamps render on
the org's wall clock (`org.update { "timezone": "Europe/Copenhagen" }`, admin or
owner; until set, the reader's browser zone, and aggregate buckets use UTC). The
clock, field order and decimal marks follow the reader's locale: a picked language,
else the browser's — but a generic `en-US` browser (headless, default installs)
follows the app's LANGUAGE, never the org's timezone: English is written the
European way (`1,234.56`, `2 Nov 2026`, `17:30`), Danish the Danish way
(`1.234,56`, `2. nov. 2026`, `17.30`). Money shows its currency's local symbol:
an English app in a Kraków org writes `1,234.56 zł`, `1,234.56 kr`, `€1,234.56`;
a Danish one `1.234,56 zł`. An empty date field shows its pattern (`dd/mm/yyyy`).
Lists, cards, chips, rails and queue rows use a compact form (`Today, 09:00`; a weekday only within
a week, `Thu 15 Oct, 15:30`; else `12 Nov, 15:30`) that wraps between day and
time rather than being cut; a record page shows the full date and time (one
shape per column: [what reads first](resources/viewspec-block-types.md#what-reads-first)).
A time bucket is named, not keyed: `Mar`, `27 Jul – 2 Aug` (a week by its days; a chart tick by
its first day), `Q1`, with the year only when the buckets are not all this year.

## Declare a view

`view_specs.declare` takes `{ name, object_type, title?, icon?, spec,
expectedHash?, force? }`; `title` and `icon` (a Lucide name) shape the nav
entry; the `title` also heads the screen over a derived summary ([page
header](resources/viewspec-block-types.md#what-reads-first)). `object_type` must equal `spec.source.objectType` for a top-level
table, detail, board or calendar (`source_object_type_mismatch`); a composite child or a
metric may bind another existing type. Declare validates against the live
registry (`unknown_object_type`) and upserts by `name` (`{ data: { name, registered: true } }`).
Outside a bundle a re-declare needs `expectedHash` (`catalog.export`'s `declaration_hash`) or
`force: true` (else `CONFLICT/HASH_REQUIRED`; stale: `CONFLICT/STALE_DECLARATION`; both:
`VALIDATION/CONTRADICTORY_DECLARATION_INTENT`). The call is the same on every surface:

```sh
canon actions invoke view_specs.declare --input @deal-pipeline.json
# REST: POST /v1/actions/view_specs.declare   body { "input": { "name": …, "object_type": …, "spec": … } }
# MCP:  tool actions.invoke   { "name": "view_specs.declare", "args": { … } }
```

## Preview before you ship

`view_specs.preview` reports what the renderer would show and writes nothing.
Input is `{ name }` (a declared view) or `{ object_type, spec }` (a dry run).

| Field | Meaning |
|---|---|
| `source.row_count` | records source: rows that would render |
| `source.value` / `source.group_count` | aggregate source: the scalar, or the number of groups |
| `columns[].status` | `resolved`, or `blanked` (would render empty); a grouped table's `count` / `sum:<p>` / dimension columns resolve by the grouped-column grammar |
| `actions[].exists` / `.visible` | does each bound action exist, and can the caller see it? |
| `context_refs[]` | `$context.*` refs that need a record in scope |
| `show_when[].field_exists` | whether each `showWhen` field exists |

An invalid spec returns `{ valid: false, errors }`. Probes run under the
**caller's** read authority — preview as the principal who will view the
screen. Loop: draft → preview (no `blanked`, counts as expected) → declare →
open the app and look.

**`showWhen` is UX, not security.** It hides a block or row action; it does not
stop the data being fetched. A `$principal.roles` / `.groups` condition is
presentation too — it hides a control from viewers it does not concern, and
grants or withholds nothing. Withholding data is server-side policy. A screen
no block covers is a Hosted/External-tier concern.
