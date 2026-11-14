---
name: canonify-springml
description: Build Canonify screens in SpringML — a `.springml` file of HTML-like JSX whose only elements are the 62 Spring UI names (Page, RecordPage, Query, Aggregate, State, Section, Table, List, Board, Stat, Chart, Form, Action, …) over data the page declares. Use it for every screen of a Canonify app; check it offline with `canon views compile`, ship it in the bundle with `canon catalog apply`.
version: 2.0.0
---

# SpringML — Canonify screens over Spring UI

A screen is one file, `canon/views/<View.name>.springml`, written like React JSX. Its only elements are the **Spring UI** catalog — 62 names, no others — and its data is **declared**, never fetched:

```jsx
{/* @catalog spring-ui */}
<Page title="Today" subtitle="Departure board · in report-time order" lead={deps}>
  <Query id="deps" type="departure" sort="report_at" page={50} nameBy="coach" current={d => d.state == "due"}>
    <Flag condition="is_late" />
  </Query>
  <Aggregate id="byState" type="departure" op="count" by="state" />
  <Grid>
    <Stat label="Out" value={byState.at("on_road")} />
    <Stat label="To go" value={byState.at("due")} context={<Text><Value of={deps.count("is_late")} /> late</Text>} />
  </Grid>
  <Section title="Departures">
    <List of={deps}>
      {d => <Item lead={d.report_at} title={d.title} eyebrow={d.trip_ref} sub={[d.coach, d.drivers]} />}
      <Action run="departure.check_out" role="primary" />
      <Action run="departure.check_in" />
      <Empty title="A quiet day">No coach leaves the depot today.</Empty>
    </List>
  </Section>
</Page>
```

That is a whole screen: a departure board with two KPI tiles, late coaches marked with a rail and "Late · 20 min", the next one marked "→ Next", the header saying "late: 07:15 SR-14 · next: 07:45 SR-05", and "Check out" on each coach that may leave. Nothing in the file says how any of it looks.

Read the base [`canonify`](../canonify/SKILL.md) skill first (auth, bundles, the authoring loop). The model a page reads — types, fields, links, named conditions, state machines, actions, policies — is declared as [`canonify-objects-actions`](../canonify-objects-actions/SKILL.md) describes; a page only **shows** it.

## How it works — the mental model

1. **The server evaluates the page, for one reader.** The page compiles to a plan; on each request the server runs the reads under that reader's policy, evaluates every expression and sends a finished tree. Nothing on the page fetches, no code you write runs in the browser, and there is no client state but the reader's choices (preset, sort, page, the States).
2. **The renderer owns every pixel.** Props say **what** something is, never how it looks: there is no `class`, `style`, `size`, `variant`, colour, width or breakpoint, no `role` or `aria-*`. Spacing, type, the card, dark mode, the phone form of every component and the format of every number, money amount and date are the platform's. A legal page is a good-looking page, and every app looks like one family.
3. **The model owns the meaning.** A business rule ("overdue", "late", "short", "mine") is a **named condition** on the type — a derived boolean with a `meaning`, an `on` field, an `amount` and a `fix` action — and the page only refers to it: `<Flag condition="is_overdue" />` gives the rail, the pill, the header count, the nav badge and the fix verb. Views narrow; they never re-derive a rule.
4. **A Query is the collection.** One server batch gives its rows, total, preset and flag counts, totals and current row, so no count on a page disagrees with its rows. Components show a Query; handles count it (`q.total`, `q.count("is_late")`); an `Aggregate` groups it (`a.at(k)`).
5. **Writes are the model's actions.** `<Action run="…" />` placed in a row, a header, a section or a toolbar; the platform builds the dialog from the action, fills its subject from where it sits, shows it only where it is legal, and follows the flag: a row shows at rest only the verb that fixes its flag. A `<Form>` is for the rare screen whose job is the form.
6. **Every reader gets their own tree.** A withheld column is removed with a note, never guessed; on a record page a withheld fact, and a section with nothing the reader may read, is left out, and `audience` on a Section or Tab places it for some groups (recipe 11); a condition over a column the reader cannot read is unknown to them (Rule C); a Query bound to a State nobody has picked does not run ("Choose a trip").

## The file

- `canon/views/<View.name>.springml`; the view name is the file stem (`Departure.today`). The **first line is `{/* @catalog spring-ui */}`** — it is what makes the file a Spring UI page. One `<Page>` or `<RecordPage type="…">` per file (a RecordPage is the record view rows of its type open).
- **Data elements first** — `Query`, `Record`, `Aggregate`, `State` — then content. Data elements are hoisted; any order works.
- Names are the model's, **exactly**: `type="departure"`, `field="hirer"` (a link by its name, not `hirer_id`), `run="charge.record_payment"`.
- Values in `{…}` are a small JavaScript subset: arrows for rows (`c => c.due_on < today()`), one-hop field access, arithmetic, `?:`, `??`, and the helpers `today()`, `now()`, `addDays`, `weekOf`, `days`, `param`, `me()`. No statements, no `new Date()`, no `.filter`/`.map`, no string building: text with values is `<Text>…<Value of={…} />…</Text>`.

## Check, then ship

SpringML needs `canon` **2026.10.26 or newer**: `canon --version` prints yours, `canon upgrade` replaces it. An older CLI answers `canon views compile` with `Unknown subcommand: views`.

```sh
canon views compile canon/views/Departure.today.springml   # offline, against the bundle around it
canon catalog validate ./canon                             # the whole bundle, as apply would
canon catalog apply ./canon --baseline <token>             # ship (the views ride in the bundle)
```

`canon views compile` needs no login: it checks the page against the catalog of the bundle the file sits in (`--canon <dir>` names another) and prints either **OK and the readback** — the tree, the read plan, each collection's resting verb, every scope fill (`departure_id ← row`) and every input left empty, the derived chrome — or **each refusal** as `file:line:col [code] problem — fix`. Fix every refusal and compile again; then read the readback: it is the screen the reader will get. Put the view in an app's `nav` (`canon/apps/<slug>.json`, its `"name"` the app's `slug` too) so people can open it, and give the page its nav glyph with `icon` (`<Page title="Payments" icon="banknote">`, a Lucide name).

`canon catalog apply` prints `counts.markup_views` (the `.springml` views it declared; `counts.views` counts legacy v1 JSON views only) and `app_urls`, one per app: `https://app.canonify.app/app/<slug>`, which opens the app's `defaultView`. A view is at `/app/<slug>/view/<View.name>` and a record page at `/app/<slug>/view/<View.name>/<record id>`; any other path is a bare "Not Found". An app is known by its `slug` (a form by its `slug`, every other artifact by its definition's `name`): a file named otherwise is applied under that key, with an `artifact_name_not_identity` warning. When the apply removed managed artifacts the bundle no longer declares, it prints them under `pruned` (`{ "apps": ["old-app"], "views": [...] }`).

## When a page is slow

A page that shows its loading skeleton for seconds has one read that costs the time. Ask the server which, as the reader who waits (their key, their State values):

```sh
canon actions invoke view_specs.evaluate --output json \
  --input '{"name":"PlantDay.plan","timing":true}' | jq '.data.timing'
```

`timing` lists every read the page made, slowest first: a data element by its `id` (`cells`, `labour`, `stock`), the reads after the plan (`$verbs` — the legality of the row verbs, `$history`, `$files`, `$lines`, `$titles.<id>`) and `$page` (the page's own lookups), each with its `stage`, `ms` (whole milliseconds), `statements` and `rows`, and the whole evaluation's `ms`. Asking for `timing` changes how the page reads: served, each stage's reads leave together in one round trip, which has no time of its own per read, so a timed page runs its reads one at a time instead. Each read's `ms` is then its own, and the whole `ms` is more than the page takes served.

A read slow for a few rows is almost always a **derived property**: its subquery runs once per row, and a derived property that names another (`load_min` over `load_kg`) repeats that subquery, so a Flag or a column over it pays again. The platform indexes the column a subquery looks rows up by when it compares it to the row (`v.date = plant_day.prev_open`) or to the clock (`v.date = :today`); a lookup written any other way — through a function (`date(v.at) = plant_day.date`) or against a subquery — scans that table for every row. Write the lookup as that comparison, and read only the derived values the page shows.

## Which component

| The screen's job | Write |
|---|---|
| A queue worked in order (a lead time, a next row, a per-row job) | `List` over a Query with `current` + `Item`, the job `Action` |
| A worklist read by its columns (sort, totals, presets) | `Table` + `Column`s, `Filters`/`Search` in a `Toolbar` |
| Lanes of a process | `Board by="status"` |
| One record | `RecordPage` with `Meta`, `Facts`, `Split`, child `Table`s, header `Action`s |
| A long record several roles read | `RecordPage` with an overview, then one titled `Section` per subject (more than three draw as tabs on their own), `audience` on the role-only ones (recipe 11) |
| KPIs and charts | `Grid` of `Stat`; `Chart shows="…"` with `Series` over an `Aggregate` |
| A checklist | `List` + `Check run undo` |
| Act on many rows | `BulkAction` in the Table |
| A builder or a wizard | `Form` with `Field`, `Step`, `Lines`, `Preview`, `Submit` |
| Resource × day; items over time; a day calendar | `Matrix` / `Timeline` / `DayGrid` |
| Assigned of needed (short / met / over), booked of capacity | `<Ratio value={…} need={…} />` / `<Ratio value={…} of={…} />` — the tone comes from the two numbers |

## Refusals you will meet first

| You wrote | Write instead |
|---|---|
| `type="Trip"` (`type_case`), `field="pasengers"` (`unknown_field`) | the model's exact name — the refusal says "did you mean" |
| `field="hirer_id"` (`id_field`) | `field="hirer"` — the link |
| `c => c.trip.status == "confirmed"` (`predicate_link`), `a \|\| b` over rows (`predicate`) | a named condition or a lookup on the type, then `c => c.trip_confirmed` |
| `className`, `style`, `tone="critical"` (`unknown_prop`) | nothing — or a named condition with a `Flag` |
| a `short` or `max` prop on a Ratio (`unknown_prop`) | `need={…}` for a requirement (below it is short) or `of={…}` for a capacity (over it is critical) |
| `<div>`, `<button>`, a name from another UI kit (`unknown_element`) | a Spring UI name — there are 62 ([components.md](resources/components.md)) |
| two Queries, no `lead` (`lead_missing`) | `lead={q}` on the Page |
| `trips.filter(…).length` (`unknown_handle`) | `trips.count("…")`, or an `Aggregate` |
| `{n + " trips"}` in Text (`expression_child`) | `<Text><Value of={n} /> trips</Text>` |
| `<Action run=… label=…>` (`action_label`) | the model's `label` / `shortLabel` |
| a Form and an Action for one action (`form_twice`) | one of them |
| two candidate records for a ref input (`fill_tie`) | `with={{trip_id: ta}}` |
| a week State without `initial` (`state_initial`) | `initial={weekOf(today())}` |
| a range State's initial `{start, end}` (`state_initial`) | `initial={{from: today(), days: 14}}` |

## Resources

| Page | Holds |
|---|---|
| [language.md](resources/language.md) | Data elements (Query, Record, Aggregate, State, Preview), the expression subset, binding to the model, withheld / Rule C / not run, the readback, refusals, caps |
| [components.md](resources/components.md) | **Generated** from the catalog schema: every one of the 62 names with its props, parents, children and an example; the helpers and handles |
| [verbs-and-forms.md](resources/verbs-and-forms.md) | `Action` placement, scope fill, legality and verbs that follow the flag, `preCheck`, `offeredWhen`, `BulkAction` over `invoke_many`, `Check`; `Form`, `Field`, `Step`, `Submit` with `next`, `Preview`, `Lines` and `seedFrom`, `Candidates`, `requiredWhen` |
| [recipes.md](resources/recipes.md) | Whole screens of the coach-hire app: today queue, payments, a list with bulk verbs, a board, a record page, a week chart, files, availability over a chosen range (busiest day, free capacity, a guard that names the day), par vs delivered over a chosen range, a need against an actual per row and day (Ratio `need`), and a long record page with section tabs and per-group sections |
