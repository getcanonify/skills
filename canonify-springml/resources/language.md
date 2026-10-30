# The SpringML language

How a page reads data, what an expression may say, how names bind to the model, and what a reader who may not see something gets. Every element and prop is in [components.md](components.md); verbs and forms are [verbs-and-forms.md](verbs-and-forms.md). Every page here compiles against the coach-hire model described at the top of components.md (each omits the `{/* @catalog spring-ui */}` first line a file needs); every refused page is refused with the code above it.

## The file

- One file, `canon/views/<View.name>.springml`, holds **one** `<Page>` or `<RecordPage>`. The view's name is the file stem (`Trip.list.springml` → `Trip.list`). The first line is `{/* @catalog spring-ui */}`; comments are `{/* … */}`.
- **Data elements first** — `Query`, `Record`, `Aggregate`, `State` sit at the top of the page (a `Query` may also sit inside an `Expand`; `Preview` sits inside its `Form`). They are hoisted: any element may read one declared later; a cycle is refused (`read_cycle`).
- **Then content** — components, and the typed parts only inside their parent (`Column` in `Table`, `Preset` in `Query`, `Field` in a `Form`, …).
- A page is **evaluated on the server for one reader**: every read runs under that reader's policy, the expressions are evaluated there, and the browser receives a finished tree. Nothing on the page fetches, and no code you write runs in the browser.

## Data elements

### Query — the collection

A `Query` is a slice of one type plus the reader's working state: the active preset, facet chips, search text, sort and page. One server batch serves it — rows, total, every preset count, every flag count and amount, column totals, groups, facets and the current row — so a count on the page can never disagree with its rows. Two components on one Query read the same results.

The [Query example](components.md#query) declares two presets and a flag and puts Filters and Search over them.

- `sort` is a field (`"-depart_date"` descending) or a list; a link sorts by its target's title; blanks sort last. `page` defaults to 25 (≤ 200; `0` = unpaged, ≤ 500). A Query a Timeline, DayGrid, Span or Matrix draws takes no `page` (refused): those have no pager, so they read it whole over what they show — the range, or the Matrix's columns — up to 500 rows, then say "Showing 500 of N". `search` defaults to the type's searchable fields (`search={false}` turns it off). `facets` names ≤ 3 enum or link fields.
- `current={d => …}` makes the Query a **queue**: the first row in sort order that matches and carries no critical flag is "→ Next".
- `Preset` is a saved view (a chip with its live count); `Flag` marks rows by a **named condition** of the type (below). ≤ 8 presets, ≤ 3 flags.
- **Handles** read the batch: `q.total`, `q.count("is_overdue")` (a preset's or any condition's row count), `q.amount("is_overdue")` (the condition's amount summed), `q.sum("open_cents")`, `q.preset`, `q.current`. Never `q.filter(…)`, `.length` or `.map` — rows are shown by a component and counted by handles.
- `lead={q}` on the Page names the collection(s) the header summary and the nav badge come from; it is **required when the page has more than one Query**. The header counts in the type's noun ("23 instalments"); a lead narrowed to a named condition counts in that condition's noun when the condition declares `label` and `labelOne`. Declare both on every condition a lead filters by, or the count reads as if it were every row (`canon views compile` warns `lead_noun_missing`): with `charge.is_overdue` declaring `"label": "overdue charges", "labelOne": "overdue charge"`, `<Page lead={od}>` over `<Query id="od" type="charge" where={c => c.is_overdue} />` reads "14 overdue charges" ("1 overdue charge").

### Record — one record by key

See the [Record example](components.md#record). A `RecordPage` reads its record from the `id` parameter and has it in scope as `record`; any other page declares its parameters with `params={{name: "type"}}` and reads them with `param("name")`. A page given a param of the wrong type (a record param that is not an id of its type, a date that is not `YYYY-MM-DD`) is refused, and a `<Link view>` or `<Action view>` may pass only the params that view declares, a record param a record of its type (`unknown_param`, `param_type`). A row opens its type's `display.recordView`, else its first RecordPage by name. A Record keyed by another Record's link adds a read stage (≤ 3 stages per page).

### Aggregate — one grouped read

`op` is `count`, `sum`, `min`, `max`, `avg` or `list`; `property` is the measured field; `by` is ≤ 2 keys (own or one-hop fields; date keys may end in `@day`, `@week`, `@month`; a week key is its ISO 8601 week, `2026-W44`, Monday to Sunday). Read a keyed Aggregate with `a.at(k)` / `a.at(k1, k2)` (missing count/sum cells are 0), an unkeyed one with `a.value`. `of={q}` inherits a Query's type and the rows it reads — its `where` and the window the components drawing it show (a Matrix's columns, a Timeline's or DayGrid's range, a week's `weekdays`), never its page, preset or search — and may only narrow them: a Stat of `<Aggregate of={cells} op="sum" …>` adds exactly the cells the Matrix draws, while an Aggregate with its own `where={d => d.date <= week.end}` also counts the Sunday a Mon–Sat week hides.

A **day series**: `over={days(week)}` makes one cell per shown day, and `span={["depart_date", "return_date"]}` counts a row on every day it covers (the day key is then written `date`):

```jsx
<Page title="Fleet week" lead={wk}>
  <State id="week" type="week" initial={weekOf(today())} />
  <Query id="wk" type="trip" page={0} where={t => t.depart_date <= week.end && t.return_date >= week.start} />
  <Aggregate id="out" type="trip" op="count" by="date" span={["depart_date", "return_date"]} over={days(week)} />
  <Aggregate id="byClass" type="trip" op="count" by={["vehicle_class", "date"]} span={["depart_date", "return_date"]} over={days(week)} />
  <Toolbar><PeriodNav state={week} span="week" /></Toolbar>
  <Grid><Stat label="On the road today" value={out.at(today())} trend={out} /></Grid>
  <Section title="On the road each day"><Chart shows="composition"><Series of={byClass} label="Trips" /></Chart></Section>
</Page>
```

`op="list"` lists the records of each cell (≤ 3 inline + a count, all of them in a popover, with `facts` per record). Caps: ≤ 5,000 cells, `over` ≤ 62 days.

### State — what the reader chooses

A State is a value the reader picks, kept in the URL under its id: `date`, `week`, `range`, `month` (each **needs** `initial`), `boolean`, `number`, `string`, `enum` (with `options`), or an ObjectType name — a **ref State**, one record or null, whose choices are its `where`/`sort` (or `from={q}`: a Query's rows), with `auto="single" | "first" | "second" | "current"` to pre-pick one. A ref State taken from the URL is re-checked on the server against its `where`. `Control` draws the picker for a State; `PeriodNav` steps a date, week, range or month ([State example](components.md#state), [Empty example](components.md#empty)). A week State has `.start` and `.end`; `days(week)` is its shown days: every day, or the `weekdays` it lists (`weekdays={["mon", "tue", "wed", "thu", "fri", "sat"]}`), which a Matrix over `days(week)`, a Timeline's day axis, a week DayGrid and PeriodNav's label and day strip all follow. A **range** State starts at `{from, days}` — its first day and its length, `initial={{from: today(), days: 14}}`, never `{start, end}` (refused, `state_initial`); it is read as `range.start` / `range.end`, kept in the URL as its first day, and PeriodNav `span="fortnight"` steps it by its length — or `span="week"` / `span="month"` by a week or a month, its length kept (a range of whole months, `{from: "2026-01-01", days: 455}`, keeps its 15 months; Today starts it this week or month). **Time grain:** a Timeline over a week, range or month State takes `grain="day" | "week" | "month"` (default day; a date State is an hour axis) — a 15-month contract timeline is `<Timeline … range={span} grain="month">` with `<PeriodNav state={span} span="month" />`; a Matrix's grain is its `cols`: `days(range)`, `weeks(range)`, `months(range)`. **Closed days:** a Matrix over days closes the days a Query names — `<Matrix cols={days(week)} … closed={off} closedKey="date" closedReason="holiday_name">` over `<Query id="off" type="calendar_day" where={c => c.is_open == false} />`: each closed column is muted, its header says "Closed · Christmas Day" and an empty cell of it "Closed" (the Query is read over the columns' days, so it needs no window term; `closedReason` defaults to the row's title). An item whose `end` is blank is open-ended: it is drawn from its start to the axis end and marked ("from 2 Mar 2026, no end"). A date, week, range or month State always has a value: an initial that gives none (a `param` the URL does not carry) fails the page naming the State, so end it with a fallback — `initial={param("date") ?? today()}`, `{from: param("from") ?? today(), days: 14}`.

### Preview — the form's live answer

Inside a `Form`, `<Preview id="pv" />` puts the action's typed preview outputs in scope as `pv.<output>` (recomputed on the server as inputs change, under the reader's policy). See [verbs-and-forms.md](verbs-and-forms.md).

## Expressions

SpringML expressions are a small JavaScript subset with two jobs.

### Row predicates — thin narrowing only

Used by Query `where` and `current`, Preset `where`, Aggregate `where`, State `where`. One arrow with one parameter:

| Form | Meaning |
|---|---|
| `r.is_overdue`, `!r.is_overdue` | a named condition of the row type |
| `r.due_on <= addDays(today(), 14)` | an own field (incl. lookups and derived) compared (`== != < <= > >=`) with a literal, a State, a Record field, a row-free helper or `param()` |
| `r.closed_on == null`, `r.trip != null` | the blank (no value, no link) as written; ordering against `null` is refused (`predicate_type`) |
| `["open", "part_paid"].includes(r.status)` | an own field in a list of literals |
| `r.trip == tr`, `r.owner == me()` | narrowing by a record (a link the row holds equals a Record or ref State); a record two links away (`r.a.b == rec`) is refused (`hops`) — declare a derived key on the row (`r.b_key == rec.id`); a to-one link whose key the other record holds is refused (`predicate_link`) — narrow by the row (`r => r == rec.<its link back>`) |
| `r == sel` | row identity |
| `a && b` | AND |
| `showAll \|\| pred` | a row-free boolean State guarding a conjunct (dropped while it is true) |

Every form is read on the server: the Query's one read narrows by it under the reader's policy, and no row is filtered after the read. A form the read cannot narrow by is refused when the page compiles.

Every **business rule lives in the model** as a named condition (a derived boolean: overdue, late, short, mine…); the view only narrows. The compiler refuses `||` between two row predicates, comparisons through a link, field against field, arithmetic or methods on row fields and ternaries, each with "declare a named condition (or a lookup) on <type>":

<!-- refused: predicate_link -->
```jsx
<Page title="Charges" lead={ch}>
  <Query id="ch" type="charge" where={c => c.trip.status == "confirmed"} />
  <Section><Table of={ch}><Column field="kind" /></Table></Section>
</Page>
```

<!-- refused: predicate -->
```jsx
<Page title="Trips" lead={trips}>
  <Query id="trips" type="trip" where={t => t.status == "quoted" || t.passengers > 40} />
  <Section><Table of={trips}><Column field="title" /></Table></Section>
</Page>
```

A blank never matches `==`, `<`, `<=`, `>`, `>=` or `includes`, and is `!=` every value; only `== null` reads it. Free-text matching is the Query's `search`, never `includes` on a text field.

### Render expressions

Used by `title`, `when`, `value`, `of`, `with`, `initial`, Item slots, Fact `value`/`sub`, Column `value`/`sub`, Cell contents. Allowed: literals, scope names, one-hop field access on rows, records and States, member access on helper results (`weekOf(today()).end`), arithmetic with the type rules below, comparisons, `&& || ! ?? ?:`, and the elements `<Text>`, `<Value>`, `<Ratio>` as values. Not allowed: locals, statements, block-bodied arrows, `.map`/`.filter`, `new`, string concatenation of numbers. Text with values in it is `<Text>` with `<Value of={…} />` inside — the platform formats every number, money amount, date and duration:

```jsx
<Page title="Today" subtitle={<Text>Departure board · now <Value of={now()} /></Text>} lead={deps}>
  <Query id="deps" type="departure" sort="report_at"><Flag condition="is_late" /></Query>
  <Section title="Late">
    <Text><Value of={deps.count("is_late")} /> of <Value of={deps.total} /> coaches are late.</Text>
  </Section>
</Page>
```

<!-- refused: expression_child -->
```jsx
<Page title="Trips" lead={trips}>
  <Query id="trips" type="trip" />
  <Text>{trips.total + " trips"}</Text>
</Page>
```

**Child functions** template a row: `{t => <Item title={t.title} … />}` in List, Board, Timeline and DayGrid; `{(row, col, cell) => <Cell>…</Cell>}` in Matrix Rows; `{t => <Section>…</Section>}` in Expand. **Row functions** give a per-row value: Column `value={c => …}` and `sub={c => …}`, Action `with={t => ({…})}`.

The helpers (`today()`, `now()`, `addDays`, `weekOf`, `monthOf`, `days`, `param`, `me()`, `greeting()`, `plural`, `percent`) and every handle are listed at the end of [components.md](components.md#helpers). There is no `new Date()`: time is the **org clock** (its zone, its demo clock), so `today()` is the same day for every reader of the org.

## Binding to the model

- **Names are the model's, exactly**: types are snake_case as declared (`type="trip"`, never `"Trip"`: `type_case`), fields, links, conditions, actions, steps, submits and preview outputs by their declared names. A misspelling is refused with "did you mean".
- **A link by its name, never its key column**: `field="hirer"`, not `hirer_id` (`id_field`). A link shows its target's title and opens its record.
- **One hop**: `t.hirer.name` / `field="trip.ref"` is fine — shown (a Column, Fact, Meta or Item slot), it is typed as the field it lands on (a date, a label, money, the linked record's title), and `field="event.client"` shows the client; two hops are refused (`hops`) — declare a lookup on the type, or read it through a Record. A Query `sort`, `facets`, a Group `by`/`measure`, a Board `by`/`sum`, a Column `sortBy` or `total` keys the read by the type's own fields, so a path there is refused (`path_key`) — declare a lookup, or key by the link itself. A to-many path in a Column shows ≤ 3 titles + "+N".
- **Types carry the look**: a money field shows its currency (minor units in the model); a number its unit; a date the one date form of its column; an enum a pill when it is the status, else its label; a condition its glyph and word. You never format. A share you compute is `percent(part, whole)` — `<Stat label="Rewash rate" value={percent(rewash.value, washed.value)} />` reads "2.9%" ("2,9 %" in Danish); never multiply by 100 yourself, which leaves a bare "2.9".
- **The language sets the convention, the zone sets the clock**: marks, dates and the currency symbol follow the app's language, never the org's timezone — an English app in a Kraków org writes `1,234.56 zł`, `1,060`, `2 Nov 2026`, `13:30`; a Danish one `1.234,56 zł`, `2. nov. 2026`, `13.30`. Titles, refusals and previews read the same way as the page.
- **Attention comes only from named conditions.** A derived boolean with a `meaning` (`overdue`, `late`, `short`, `over`, `blocked` → critical; `at_risk`, `due_soon`, `full` → warn; `met` → good), an `on` field (for the distance: "Late · 20 min"), an `amount` and a `fix` action. `<Flag condition="is_late" />` on a Query is the whole instruction: the bar, the pill, the count in the header, the nav badge and the fix verb follow from the condition. There is no `tone`, `color` or `variant` anywhere (a Callout is `info` or `warn`, nothing else).
- **The platform owns the look**: no `class`, `style`, `size`, `width`, breakpoint, `role`, `aria-*` or key handler; nesting is semantic (a top-level Section is one card; depth sets heading levels). Phone and narrow forms are each component's own.

## Who sees what: withheld, Rule C, not run

A page is one tree **per reader**. What a reader may not read is never sent to them, and is never guessed:

- **Withheld columns.** A Column whose field the reader's policy withholds is removed, with one header note ("Price hidden for your role"); a Stat, a single Value or a Page's Fact shows "Hidden for your role". A record page serves each reader their part of the record: a Fact they may not read is left out of its Facts or Meta, and a Facts, Section or Tab with nothing left they may read is not served (a Split left with one side gives it the whole width) — never a column of lock rows. `audience={["sales"]}` on a Section or Tab places it for those groups only (owners and admins see every one); others are not served it at all (recipe 11). A Preset, facet, Flag, Group or sort that needs a withheld column is dropped for that reader and its count shows as withheld. A derived value, condition, rollup or preview output computed from a withheld column is withheld too. A `when` with a withheld operand is false.
- **Rule C — a guard reveals only when it says so.** A condition or guard that reads a column the reader cannot see is unknown to that reader: a verb `offeredWhen` such a condition is offered on no row alike, and a guard over it refuses uniformly, whatever the row holds — unless the model declares that guard `reveals: true`. The page cannot change this, and need not: write the verb; each reader gets the verbs their policy allows.
- **Not run — "Choose …".** A predicate that compares with a ref State or Record that is null, or with a State not chosen yet, does not run its Query (it does not read "every row"): its components show "Choose <noun>", or the first `<Empty when={…}>` that matches. A Section whose `of` is null hides its record title and the components that read it. The [Empty example](components.md#empty) is a page that waits for its trip.
- **Views and records.** A row opens its type's record view — a `RecordPage` of that type (with several, the first by view name; `opens="View.name"` or `opens="none"` says otherwise) — only when the reader may open it. `audience` on a Page hides the view from other groups in nav and links. A page that is a Form (its Forms run actions) leads to its action: a Link, an `<Action view>` or a nav entry to it is offered only to a reader who may run one of them — a role the action refuses never sees "New contract" lead to "not available for your role". The nav names a page by its `navLabel` (literal text), else its title when that is text, else the last part of the view name (`Sales.dashboard` → "Dashboard"); a reader who sees two entries with the same name, as an owner sees every group's page, gets each followed by its audience ("Dashboard · Sales"), else its page name — so give pages that share a title a `navLabel` when the audience is not the difference. The nav draws a page with its `icon` (`<Page icon="banknote">`, a Lucide name from lucide.dev/icons; an unknown name is refused with close ones), else its kind's glyph (a record page, a board, a timeline, a form, a table…) — give each nav page its own; an app's nav entry takes no `icon`. A page about a record — a `RecordPage`, or a Page whose title names a record — is named by its title in the app's breadcrumb, the phone's top bar and the browser tab (a reader whose policy withholds the title reads the type's noun, "Trip").

## Compile, read back, fix

`canon views compile canon/views/<View.name>.springml` checks a page against the catalog in the bundle around it (or `--canon <dir>`), offline: every element, prop, name and expression, and the caps. On success it prints the **readback** — the tree, the resting verb of each collection, every scope fill ("`departure_id ← row`") and every input left empty by rule, the read plan with its stages, and the derived chrome (header summary, flagged rows, the queue's "→ Next"). Read it: it is how you see what the reader will get. On failure each refusal names `file:line:col`, a code, the problem and the form to write:

<!-- refused: id_field -->
```jsx
<Page title="Trips" lead={trips}>
  <Query id="trips" type="trip" />
  <Section><Table of={trips}><Column field="hirer_id" /></Table></Section>
</Page>
```

<!-- refused: unknown_handle -->
```jsx
<Page title="Trips" lead={trips}>
  <Query id="trips" type="trip" />
  <Grid><Stat label="Quoted" value={trips.filter(t => t.status == "quoted").length} /></Grid>
</Page>
```

<!-- refused: unknown_prop -->
```jsx
<Page title="Trips" lead={trips}>
  <Query id="trips" type="trip" />
  <Section><Table of={trips} where={t => t.status == "quoted"}><Column field="title" /></Table></Section>
</Page>
```

Other refusals you will meet: `lead_missing` (two Queries and no `lead`), `state_initial` (a date, week, range or month State without `initial`, or with one of the wrong shape: a range's is `{from, days}`), `unknown_element` (a name Spring UI does not have — there are 62), `unknown_field` / `unknown_condition` / `unknown_action` (with "did you mean"), `hops`, `path_key` (a sort, facet, group, lane or total through a link), `id_shadowing` (an id named like a helper: `days`, `today`), `aggregate_arity` (`.at()` with the wrong number of keys), `parent` / `needs_ancestor` (a part outside its parent), `parent_rule` (a Query inside a Section that is not in an Expand), `search_off`, `board_by` (Board lanes must be an enum or machine field), and the verb refusals of [verbs-and-forms.md](verbs-and-forms.md). A refusal that says **"requires PA-n"** names a model feature the catalog cannot declare yet: write the page without that part.

## Caps

Query: page ≤ 200 (unpaged ≤ 500), presets ≤ 8, flags ≤ 3, totals ≤ 16, facets ≤ 3. Aggregate: ≤ 2 keys, ≤ 5,000 cells, `over` ≤ 62 days. Matrix ≤ 500 rows (one whole read) × 31 day, 26 week or 24 month columns; Timeline ≤ 100 lanes over ≤ 31 days, 26 weeks or 24 months by `grain`, ≤ 500 items (past a period cap both draw the first periods and say so: "Showing the first 26 of 30 weeks"); DayGrid ≤ 20 lanes; a Board takes every state of its machine (the lanes scroll, and its terminal and sink states fold into one lane with a count per state that opens on demand — each state still a drop target); Table ≤ 8 columns of importance primary, eyebrow or secondary (mark the rest `detail`); Meta ≤ 6 Facts; ≤ 3 header verbs offered together; ≤ 3 read stages per page.
