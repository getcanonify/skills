---
name: canonify-springml
description: Build Canonify screens in SpringML — a `.springml` file of HTML-like JSX whose only elements are the Spring UI names (Page, RecordPage, Query, Aggregate, State, Section, Table, List, Stat, Chart, Form, Action, …) over data the page declares. Use it for every screen of a Canonify app; check it offline with `canon views compile`, ship it in the bundle with `canon catalog apply`.
version: 2.0.0
---

# SpringML — Canonify screens over Spring UI

A screen is one file, `canon/views/<view_name>.springml`, written like React JSX. Its only elements are the **Spring UI** catalog — 62 names, no others — and its data is **declared**, never fetched:

```jsx
{/* @catalog spring-ui */}
<Page title="Hold shelf" subtitle="Holds ready for pick-up · oldest pick-up time first" lead={holds}>
  <Query id="holds" type="hold" sort="pickup_by" page={50} nameBy="shelf" current={h => h.state == "ready"}>
    <Flag condition="is_expiring" />
  </Query>
  <Aggregate id="byState" type="hold" op="count" by="state" />
  <Grid>
    <Stat label="Handed over" value={byState.at("collected")} />
    <Stat label="Waiting" value={byState.at("ready")} context={<Text><Value of={holds.count(h => h.is_expiring)} /> past their pick-up time</Text>} />
  </Grid>
  <Section title="Holds">
    <List of={holds}>
      {h => <Item lead={h.pickup_by} title={h.item} eyebrow={h.shelf} sub={[h.patron_name, h.card_no]} />}
      <Action run="hold_collect" role="primary" />
      <Action run="hold_extend" />
      <Empty title="An empty shelf">No hold is waiting to be picked up.</Empty>
    </List>
  </Section>
</Page>
```

That is a whole screen: a library's hold shelf with two KPI tiles, the holds whose pick-up time has passed marked with a rail and a "pick-up time passed" pill, the next one marked "→ Next", the header saying "pick-up time passed: 27 Nov, 07:00 H-03 · next: 29 Nov, 18:00 H-07", and "Hand over" on each hold that may go. Nothing in the file says how any of it looks.

Read the base [`canonify`](../canonify/SKILL.md) skill first (auth, bundles, the authoring loop). The model a page reads — types, fields, links, named conditions, state machines, actions, policies — is declared as [`canonify-objects-actions`](../canonify-objects-actions/SKILL.md) describes; a page only **shows** it.

## How it works — the mental model

1. **The server evaluates the page, for one reader.** The page compiles to a plan; on each request the server runs the reads under that reader's policy, evaluates every expression and sends a finished tree. Nothing on the page fetches, no code you write runs in the browser, and there is no client state but the reader's choices (preset, sort, page, the States).
2. **The renderer owns every pixel.** Props say **what** something is, never how it looks: there is no `class`, `style`, `size`, `variant`, colour, width or breakpoint, no `role` or `aria-*`. Spacing, type, the card, dark mode, the phone form of every component and the format of every number, money amount and date are the platform's. A legal page is a good-looking page, and every app looks like one family.
3. **The model owns the meaning.** A business rule ("overdue", "expiring", "short", "mine") is a **named condition** on the type — a derived boolean the type's `presentation.conditions` gives a `level` (critical, warn, good, info, neutral), an `on` field, an `amount` and a `fix` action — and the page only refers to it: `<Flag condition="is_overdue" />` gives the rail, the pill, the header count, the nav badge and the fix verb. Views narrow; they never re-derive a rule.
4. **A Query is the collection.** One server batch gives its rows, total, preset and flag counts, totals and current row, so no count on a page disagrees with its rows. Components show a Query; its methods count it (`q.count()`, `q.count(r => r.is_overdue)`, `q.sum(r => r.open_cents)`), never a `q.total`; an `Aggregate` groups it (`a.at(k)`).
5. **Writes are the model's actions.** `<Action run="…" />` placed in a row, a header, a section or a toolbar; the platform builds the dialog from the action, fills its subject from where it sits (a row, a Section's `of`, a RecordPage), offers it where it is legal and draws a refused one as a line with its reason, and follows the flag: a row shows at rest only the verb that fixes its flag. A `<Form>` is for the rare screen whose job is the form.
6. **Every reader gets their own tree.** A withheld column is removed with a note, never guessed; on a record page a withheld fact, and a section with nothing the reader may read, is left out, and `audience` on a Section or Tab places it for some groups (recipe 1); a condition over a column the reader cannot read is unknown to them (Rule C); a Query bound to a State nobody has picked does not run ("Choose a patron").

## The file

- `canon/views/<view_name>.springml`; the view name is the file stem, in snake_case as every model name (`holds_today`). The **first line is `{/* @catalog spring-ui */}`** — it is what makes the file a Spring UI page. One `<Page>` or `<RecordPage type="…">` per file (a RecordPage is the record view rows of its type open).
- **Data elements first** — `Query`, `Record`, `Aggregate`, `State` — then content. Data elements are hoisted; any order works.
- Names are the model's, **exactly**: `type="hold"`, `field="patron"` (a link by its name, not `patron_id`), `run="fine_record_payment"`.
- Values in `{…}` are a small JavaScript subset: arrows for rows (`c => c.due_on < today()`), one-hop field access, arithmetic, `?:`, `??`, and the helpers `today()`, `now()`, `addDays`, `weekday`, `yearsBetween`, `weekOf`, `days`, `param`, `me()`. No statements, no `new Date()`, no `.filter`/`.map`, no string building: text with values is `<Text>…<Value of={…} />…</Text>`.

## Check, then ship

SpringML needs `canon` **2026.10.26 or newer**: `canon --version` prints yours, `canon upgrade` replaces it. An older CLI answers `canon views compile` with `Unknown subcommand: views`.

```sh
canon views compile canon/views/holds_today.springml      # offline, against the bundle around it
canon catalog validate ./canon                             # the whole bundle, as apply would
canon catalog apply ./canon --baseline <token>             # ship (the views ride in the bundle)
```

`canon views compile` needs no login: it checks the page against the catalog of the bundle the file sits in (`--canon <dir>` names another) and prints either **OK and the readback** — the tree, the read plan, each collection's resting verb, every scope fill (`hold_id ← row`) and every input left empty, the derived chrome — or **each refusal** as `file:line:col [code] problem — fix`. Fix every refusal and compile again; then read the readback: it is the screen the reader will get. Put the view in an app's `nav` (`canon/apps/<slug>.json`, its `"name"` the app's `slug` too) so people can open it, and give the page its nav glyph with `icon` (`<Page title="Fines" icon="banknote">`, a Lucide name; it compiles, not drawn yet: K6.17).

`canon catalog apply` prints `counts.markup_views` (the `.springml` views it declared; `counts.views` counts legacy v1 JSON views only) and `app_urls`, one per app: `https://app.canonify.app/app/<slug>`, which opens the app's `defaultView`. A view is at `/app/<slug>/view/<View.name>` and a record page at `/app/<slug>/view/<View.name>/<record id>`; any other path is a bare "Not Found". An app is known by its `slug` (a form by its `slug`, every other artifact by its definition's `name`): a file named otherwise is applied under that key, with an `artifact_name_not_identity` warning. When the apply removed managed artifacts the bundle no longer declares, it prints them under `pruned` (`{ "apps": ["old-app"], "views": [...] }`).

## When a page is slow

A page that shows its loading skeleton for seconds has one read that costs the time. Ask the server which, as the reader who waits (their key, their State values):

```sh
canon actions invoke view_specs.evaluate --output json \
  --input '{"name":"room_days","timing":true}' | jq '.data.timing'
```

`timing` lists every read the page made, slowest first: a data element by its `id` (`roomDays`, `booked`, `extra`), the reads after the plan (`$verbs` — the legality of the row verbs, `$history`, `$files`, `$lines`, `$titles.<id>`) and `$page` (the page's own lookups), each with its `stage`, `ms` (whole milliseconds), `statements` and `rows`, and the whole evaluation's `ms`. Asking for `timing` changes how the page reads: served, each stage's reads leave together in one round trip, which has no time of its own per read, so a timed page runs its reads one at a time instead. Each read's `ms` is then its own, and the whole `ms` is more than the page takes served.

A read slow for a few rows is almost always a **derived property**: its subquery runs once per row, and a derived property that names another (`owed_cents` over `open_cents`) repeats that subquery, so a Flag or a column over it pays again. The platform indexes the column a subquery looks rows up by when it compares it to the row (`b.date = room_day.date`) or to the clock (`b.date = :today`); a lookup written any other way — through a function (`date(b.starts_at) = room_day.date`) or against a subquery — scans that table for every row. Write the lookup as that comparison, and read only the derived values the page shows.

## Which component

| The screen's job | Write |
|---|---|
| A queue worked in order (a lead time, a next row, a per-row job) | `List` over a Query with `current` + `Item`, the job `Action` |
| A worklist read by its columns (sort, totals, presets) | `Table` + `Column`s, `Filters`/`Search` in a `Toolbar` |
| One record | `RecordPage` with `Meta`, `Facts`, `Split`, child `Table`s, header `Action`s |
| A long record several roles read | `RecordPage` with an overview, then one titled `Section` per subject (more than three draw as tabs on their own), `audience` on the role-only ones (recipe 1) |
| KPIs and charts | `Grid` of `Stat`; `Chart shows="…"` with `Series` over an `Aggregate` |
| A checklist | `List` + `Check run undo` |
| Act on many rows | `BulkAction` in the Table |
| A builder or a wizard | `Form` with `Field`, `Step`, `Lines`, `Preview`, `Submit` |
| A screen whose layout is the data: a calendar, a schedule, a resource × time grid, a board of nested marks | a `Region` your code draws ([regions.md](resources/regions.md)) |
| Assigned of needed (short / met / over), booked of capacity | `<Ratio value={…} need={…} />` / `<Ratio value={…} of={…} />` — the tone comes from the two numbers |

## When the layout is the data: a Region

A record, a list, a form or a dashboard is a document: write it with the names above. A calendar, a schedule, a resource × time grid or a board of nested marks is a program: there **where a mark sits is the data**. Declare its data as usual (Queries, Aggregates, States, verbs). Then hand it to your own code with one element, `<Region code="week" data={{bookings: bookings}} range={week} />`. The module `views/<view>/week.region.js` draws that one region over the same per-reader model the platform's components get: rows, withheld values, legal verbs and `mine`, all decided on the server. Its verbs are the Region's `Action` children, and it runs them only through `platform.open(verb)`. It draws with @forspring/ui's tokens and classes, in a frame with no network or storage. The checks (`tokens-only`, `no-network`, `visibility`, `overflow-390`, `basics`) run in the author loop and at `catalog.apply`. Everything else on the page stays SpringML. [regions.md](resources/regions.md) has the model, the calls, an example and what each check refuses.

## Refusals you will meet first

| You wrote | Write instead |
|---|---|
| `type="Loan"` (`page.bind.unknown_type`), `field="renewls"` (`page.bind.unknown_field`) | the model's exact name — the refusal says "did you mean" |
| `field="patron_id"` (`page.bind.id_field`) | `field="patron"` — the link |
| `l => l.patron.status == "active"` (`page.bind.link_comparison`), `a \|\| b` over rows (`page.bind.or_of_rows`) | a named condition or a lookup on the type, then `l => l.patron_active` |
| `className`, `style`, `tone="critical"` (`page.structure.unknown_prop`) | nothing — or a named condition with a `Flag` |
| `<Flag condition="is_late" />` on a converted catalog (`page.bind.unknown_condition`: it declares no conditions) | declare the condition in your bundle: `presentation.conditions` on the type, keyed by a derived boolean (add it if missing), with a `level` ([object-property-types.md](../canonify-objects-actions/resources/object-property-types.md#named-conditions--presentationconditions)) |
| a `short` or `max` prop on a Ratio (`page.structure.unknown_prop`) | `need={…}` for a requirement (below it is short) or `of={…}` for a capacity (over it is critical) |
| `<div>`, `<button>`, a name from another UI kit (`page.structure.unknown_element`) | a Spring UI name ([components.md](resources/components.md)) |
| two Queries, no `lead` (`page.bind.lead_many`) | `lead={q}` on the Page |
| `loans.filter(…).length`, `loans.total` (`cx.name.unknown`, `page.bind.collection_total`) | `loans.count()`, `loans.count(l => l.is_overdue)`, or an `Aggregate` |
| `{n + " loans"}` in Text (`page.structure.expression_child`) | `<Text><Value of={n} /> loans</Text>` |
| `<Action run=… label=…>` (`page.bind.action_label_run`) | the model's `label` / `shortLabel` |
| a Form and an Action for one action (`page.bind.form_twice`) | one of them |
| two candidate records for a ref input (`page.bind.fill_tie`) | `with={{patron_id: pt}}` |
| a week State without `initial` (`page.bind.state_initial`) | `initial={weekOf(today())}` |
| a range State's initial `{from, days}` or `{start, end}` (`page.bind.period_initial`) | `initial={period(today(), addDays(today(), 13))}` |

## Resources

| Page | Holds |
|---|---|
| [language.md](resources/language.md) | Data elements (Query, Record, Aggregate, State, Preview), the expression subset, binding to the model, withheld / Rule C / not run, the readback, refusals, caps |
| [components.md](resources/components.md) | **Generated** from the catalog schema: every name an author writes, with its props, parents, children and an example; the helpers and handles |
| [verbs-and-forms.md](resources/verbs-and-forms.md) | `Action` placement, scope fill, legality and verbs that follow the flag, `preCheck`, `offeredWhen`, `BulkAction` over `invoke_many`, `Check`; `Form`, `Field`, `Step`, `Submit` with `next`, `Preview`, `Lines` and `seedFrom`, `Candidates`, `requiredWhen` |
| [recipes.md](resources/recipes.md) | Whole screens of a city library (and a sports club): the hold shelf, fines, a list with bulk verbs, a record page, a week chart, files, availability over a chosen range (busiest day, free capacity, a guard that names the day), par vs delivered over a chosen range, a need against an actual per row and day (Ratio `need`), and a long record page with section tabs and per-group sections |
| [regions.md](resources/regions.md) | A screen whose layout is the data: the `Region` element and its module, the per-reader model it draws, `platform.open` for verbs, the look and the checks |
