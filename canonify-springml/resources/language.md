# The SpringML language

How a page reads data, what an expression may say, how names bind to the model, and what a reader who may not see something gets. Every element and prop is in [components.md](components.md); verbs and forms are [verbs-and-forms.md](verbs-and-forms.md). Every page here compiles against the city library model described at the top of components.md (each omits the `{/* @catalog spring-ui */}` first line a file needs); every refused page is refused with the code above it.

## The file

- One file, `canon/views/<view_name>.springml`, holds **one** `<Page>` or `<RecordPage>`. The view's name is the file stem (`loan_list.springml` → `loan_list`). The first line is `{/* @catalog spring-ui */}`; comments are `{/* … */}`.
- **Data elements first** — `Query`, `Record`, `Aggregate`, `State` sit at the top of the page (a `Query` may also sit inside an `Expand`; `Preview` sits inside its `Form`). They are hoisted: any element may read one declared later; a cycle is refused (`read_cycle`).
- **Then content** — components, and the typed parts only inside their parent (`Column` in `Table`, `Preset` in `Query`, `Field` in a `Form`, …).
- A page is **evaluated on the server for one reader**: every read runs under that reader's policy, the expressions are evaluated there, and the browser receives a finished tree. Nothing on the page fetches, and no code you write runs in the browser.

## Data elements

### Query — the collection

A `Query` is a slice of one type plus the reader's working state: the active preset, facet chips, search text, sort and page. One server batch serves it — rows, total, every preset count, every flag count and amount, column totals, groups, facets and the current row — so a count on the page can never disagree with its rows. Two components on one Query read the same results.

The [Query example](components.md#query) declares two presets and a flag and puts Filters and Search over them.

- `sort` is a field (`"-due_on"` descending) or a list; a link sorts by its target's title; blanks sort last. `page` defaults to 25 (≤ 200; `0` = unpaged, ≤ 500). `search` defaults to the type's searchable fields (`search={false}` turns it off). `facets` names ≤ 3 enum or link fields.
- `current={d => …}` makes the Query a **queue**: the first row in sort order that matches and carries no critical flag is "→ Next".
- `mine={r => r.holder == me()}` marks the reader's own rows (`RowView.mine`, never a reader id): a row predicate under the rules of `where` (a person through a link: a declared lookup, `over_readable`, compared with `me()`); absent where it is false, blank or withheld.
- `Preset` is a saved view (a chip with its live count); `Flag` marks rows by a **named condition** of the type (below). ≤ 8 presets, ≤ 3 flags.
- **Handles** read the batch: `q.count()`, `q.count(r => r.is_overdue)` (a preset's or any condition's row count), `q.where(r => r.is_overdue).sum(r => r.open_cents)` (the condition's amount summed), `q.sum(r => r.open_cents)`, `q.preset`. They count and sum the rows the reader is **shown** (the Query's read under the reader's policy); a count or sum that reads a column the reader's policy masks is **withheld**, never a total over the rest. Never `q.filter(…)`, `.length` or `.map` — rows are shown by a component and counted by handles.
- `lead={q}` on the Page names the collection(s) the header summary and the nav badge come from; it is **required when the page has more than one Query**. The header counts in the type's noun ("23 loans").

### Record — one record by key

See the [Record example](components.md#record). A `RecordPage` reads its record from the `id` parameter and has it in scope as `record`; any other page declares its parameters with `params={{name: "type"}}` and reads them with `param("name")`. A page given a param of the wrong type (a record param that is not an id of its type, a date that is not `YYYY-MM-DD`) is refused, and a `<Link view>` or `<Action view>` may pass only the params that view declares, a record param a record of its type (`unknown_param`, `param_type`). A row opens its type's `display.recordView`, else its first RecordPage by name. A Record keyed by another Record's link adds a read stage (≤ 3 stages per page). **Eyebrow.** `Page` and `RecordPage` take `eyebrow={expression}` (`eyebrow={record.code}`): a quiet label above the title. Ids and codes (an order number, a booking reference) go in the eyebrow, never in the display `title`. The reader's rights apply as to any value: a blank or withheld one leaves the eyebrow out, with no placeholder. It is a prop, not an element; it takes an expression, so `eyebrow="Sales"` is refused (write `eyebrow={"Sales"}`).

### Aggregate — one grouped read

`op` is `count`, `sum`, `min`, `max`, `avg` or `list`; `property` is the measured field; `by` is ≤ 2 keys (own or one-hop fields; a date field may end in `@week`, `@month`; a week key is its ISO 8601 week, `2026-W44`, Monday to Sunday). A timestamp is not a date: to count by day, declare a derived date on the type (`ready_on`: `dateOf(ready_at)`) and key by it; compare it, not the timestamp, with a date (`h.ready_on == today()`). A planned finish is a derived timestamp: `at(day, start) + duration(hours, 'h')` (`hours` may be a decimal, 6.5; the sum is elapsed time, so a wall clock across a DST change jumps). Read a keyed Aggregate with `a.at(k)` / `a.at(k1, k2)` (missing count/sum cells are 0), an unkeyed one with `a.value`. `of={q}` inherits a Query's type and the rows it reads — its `where` (a week's `weekdays` too), never its page, preset or search — and may only narrow them: a Stat of `<Aggregate of={days} op="sum" …>` adds exactly the rows the Query reads, while an Aggregate with its own `where={d => d.date <= week.end}` also counts the Sunday a Mon–Sat week hides.

A **day series**: `over={days(week)}` makes one cell per shown day of a date key. A row that covers several days (a loan from the day it goes out to the day it is due, a booking over a weekend) counts on each of them only through a row per day: `span` is **not drawn yet** (refused, `page.structure.unsupported`), so model the per-day row (`room_day`: a room × date with `booked`) and sum it:

```jsx
<Page title="Study rooms this week" lead={wk}>
  <State id="week" type="week" initial={weekOf(today())} />
  <Query id="wk" type="room_day" page={0} where={d => d.date >= week.start && d.date <= week.end} />
  <Aggregate id="seats" type="room_day" op="sum" property="booked" by="date" over={days(week)} />
  <Aggregate id="byRoom" type="room_day" op="sum" property="booked" by={["room", "date"]} over={days(week)} />
  <Toolbar><PeriodNav state={week} span="week" /></Toolbar>
  <Grid><Stat label="Seats booked today" value={seats.at(today())} trend={seats} /></Grid>
  <Section title="Seats booked each day"><Chart shows="composition"><Series of={byRoom} label="Seats" /></Chart></Section>
</Page>
```

`op="list"` lists the records of each cell (≤ 3 inline + a count, all of them in a popover, with `facts` per record). Caps: ≤ 5,000 cells, `over` ≤ 62 days.

### State — what the reader chooses

A State is a value the reader picks, kept in the URL under its id: `date`, `week`, `range`, `month` (each **needs** `initial`), `boolean`, `number`, `string`, `enum` (with `options`), or an ObjectType name — a **ref State**, one record or null, whose choices are its `where`/`sort` (or `from={q}`: a Query's rows), picked by the reader (`auto`, which would pre-pick one, is **not drawn yet**: refused). A ref State taken from the URL is re-checked on the server against its `where`. `Control` draws the picker for a State; `PeriodNav` steps a date, week, range or month ([State example](components.md#state), [Empty example](components.md#empty)). A week State has `.start` and `.end`; `days(week)` is its shown days: every day, or the `weekdays` it lists (`weekdays={["mon", "tue", "wed", "thu", "fri", "sat"]}`), which PeriodNav's label and day strip follow. A **range** State starts at `period(from, to)` — its first and last day, `initial={period(today(), addDays(today(), 13))}`, never an object such as `{from, days}` (refused, `page.bind.period_initial`); it is read as `range.start` / `range.end`, kept in the URL as its first day, and PeriodNav `span="fortnight"` steps it by its length — or `span="week"` / `span="month"` by a week or a month, its length kept (a range of whole months, `period(date("2026-01-01"), date("2027-03-31"))`, keeps its 15 months; Today starts it this week or month). A date, week, range or month State always has a value: an initial that gives none (a `param` the URL does not carry) fails the page naming the State, so end it with a fallback — `initial={param("date") ?? today()}`, `period(param("from") ?? today(), addDays(param("from") ?? today(), 13))`.

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
| `isBlank(r.returned_on)`, `isSet(r.loan)` | the blank (no value, no link); `r.returned_on == null` is refused (`null_compare`): write isBlank / isSet; ordering against a blank is refused (`predicate_type`) |
| `r.status in ["open", "part_paid"]` | an own field in a list of literals |
| `r.patron == pt`, `r.owner == me()` | narrowing by a record (a link the row holds equals a Record or ref State); a record two links away (`r.a.b == rec`) is refused (`hops`) — declare a derived key on the row (`r.b_key == rec.id`); a to-one link whose key the other record holds is refused (`predicate_link`) — narrow by the row (`r => r == rec.<its link back>`) |
| `r == sel` | row identity |
| `a && b` | AND |
| `showAll \|\| pred` | a row-free boolean State guarding a conjunct (dropped while it is true) |

Every form is read on the server: the Query's one read narrows by it under the reader's policy, and no row is filtered after the read. A form the read cannot narrow by is refused when the page compiles.

Every **business rule lives in the model** as a named condition (a derived boolean: overdue, late, short, mine…); the view only narrows. The compiler refuses `||` between two row predicates (`page.bind.or_of_rows`) and a comparison through a link (`page.bind.link_comparison`); a rule written as field against field, arithmetic or a ternary in a `where` compiles but belongs in the model too. A quantity field compares with a quantity (`r.weight > quantity(20, "kg")`), never a bare number:

<!-- refused: page.bind.link_comparison(patron.status) -->
```jsx
<Page title="Loans" lead={loans}>
  <Query id="loans" type="loan" where={l => l.patron.status == "active"} />
  <Section><Table of={loans}><Column field="item" /></Table></Section>
</Page>
```

<!-- refused: page.bind.or_of_rows() -->
```jsx
<Page title="Loans" lead={loans}>
  <Query id="loans" type="loan" where={l => l.state == "out" || l.renewals > 2} />
  <Section><Table of={loans}><Column field="item" /></Table></Section>
</Page>
```

A blank never matches `==`, `<`, `<=`, `>`, `>=` or `in`, and is `!=` every value; only `isBlank(…)` reads it. Free-text matching is the Query's `search`, never `includes` on a text field.

### Render expressions

Used by `title`, `when`, `value`, `of`, `with`, `initial`, Item slots, Fact `value`/`sub`, Column `value`/`sub`. Allowed: literals, scope names, one-hop field access on rows, records and States, member access on helper results (`weekOf(today()).end`), arithmetic with the type rules below, comparisons, `&& || ! ?? ?:`. The elements `<Text>`, `<Value>`, `<Ratio>` are values only where a prop takes an element (a Page or Section `title`, a Page or RecordPage `subtitle`, a Stat `context`, a Fact `value` or `sub`, a Column `value` or `sub` row function); a ternary picks between values, never between elements: a phrase shown on a condition is `<Text when={c}>…</Text>`. Not allowed: locals, statements, block-bodied arrows, `.map`/`.filter`, `new`, string concatenation of numbers. Text with values in it is `<Text>` with `<Value of={…} />` inside — the platform formats every number, money amount, date and duration:

```jsx
<Page title="Hold shelf" subtitle={<Text>Ready for pick-up · now <Value of={now()} /></Text>} lead={holds}>
  <Query id="holds" type="hold" sort="pickup_by"><Flag condition="is_expiring" /></Query>
  <Section title="Past their pick-up time">
    <Text><Value of={holds.count(h => h.is_expiring)} /> of <Value of={holds.count()} /> holds are past their pick-up time.</Text>
  </Section>
</Page>
```

<!-- refused: page.structure.expression_child(Text) -->
```jsx
<Page title="Loans" lead={loans}>
  <Query id="loans" type="loan" />
  <Text>{loans.count() + " loans"}</Text>
</Page>
```

**Child functions** template a row: `{l => <Item title={l.item} … />}` in List; `{t => <Section>…</Section>}` in Expand. **Row functions** give a per-row value: Column `value={c => …}` and `sub={c => …}`, Action `with={t => ({…})}`.

The helpers (`today()`, `now()`, `addDays`, `weekday`, `yearsBetween`, `weekOf`, `monthOf`, `days`, `param`, `me()`, `greeting()`, `plural`, `percent`) and every handle are listed at the end of [components.md](components.md#helpers). There is no `new Date()`: time is the **org clock** (its zone, its demo clock), so `today()` is the same day for every reader of the org.

## Binding to the model

- **Names are the model's, exactly**: types are snake_case as declared (`type="loan"`, never `"Loan"`: `type_case`), fields, links, conditions, actions, steps, submits and preview outputs by their declared names. A misspelling is refused with "did you mean".
- **A link by its name, never its key column**: `field="patron"`, not `patron_id` (`id_field`). A link shows its target's title and opens its record.
- **One hop**: `l.patron.name` / `field="loan.item"` is fine — shown (a Column, Fact, Meta or Item slot), it is typed as the field it lands on (a date, a label, money, the linked record's title), and `field="copy.branch"` shows the branch; two hops are refused (`hops`) — declare a lookup on the type, or read it through a Record. A Query `sort`, `facets`, a Group `by`/`measure` a Column `sortBy` or `total` keys the read by the type's own fields, so a path there is refused (`path_key`) — declare a lookup, or key by the link itself. A to-many path in a Column shows ≤ 3 titles + "+N".
- **Types carry the look**: a money field shows its currency (minor units in the model); a number its unit; a date the one date form of its column; an enum a pill when it is the status, else its label; a condition its glyph and word. You never format; whole amounts are `decimals={0}` on Fact, Column, Item, Value or Ratio ("377 kr", not "377.00 kr"). A share you compute is `percent(part, whole)` — `<Stat label="Renewal rate" value={percent(renewed.value, loaned.value)} />` reads "2.90 %"; never multiply by 100 yourself, which leaves a bare "2.9".
- **The language sets the convention, the zone sets the clock**: marks, dates and the currency symbol follow the app's language, never the org's timezone — an English app in an org on a Zurich clock writes `CHF 1,234.56`, `1,060`, `2 Nov 2026`, `13:30`; a Danish one `1.234,56 CHF`, `2. nov. 2026`, `13.30`. Titles, refusals and previews read the same way as the page.
- **Attention comes only from named conditions.** A derived boolean the type's `presentation.conditions` names, keyed by that property, with a `level` (`critical`, `warn`, `good`, `info` or `neutral`: its tone), a `meaning` (`critical`, `warn` or `good`), an `on` property (for the distance: "Late · 20 min"), an `amount` property and a `fix` action; its words are the property's label ([canonify-objects-actions](../../canonify-objects-actions/resources/object-property-types.md#named-conditions--presentationconditions) declares it). A converted catalog declares none: declare the ones your screens need in your own bundle, or every `Flag` is refused (`page.bind.unknown_condition`). `<Flag condition="is_overdue" />` on a Query is the whole instruction: the bar, the pill, the count in the header, the nav badge and the fix verb follow from the condition. There is no `tone`, `color` or `variant` anywhere (a Callout's `tone` is `info`, `warn` or `critical`, nothing else: critical is the red of a critical status, and a Callout tied to a critical condition draws critical).
- **The platform owns the look**: no `class`, `style`, `size`, `width`, breakpoint, `role`, `aria-*` or key handler; nesting is semantic (a top-level Section is one card; depth sets heading levels). Phone and narrow forms are each component's own.

## Who sees what: withheld, Rule C, not run

A page is one tree **per reader**. What a reader may not read is never sent to them, and is never guessed:

- **Withheld columns.** A Column whose field the reader's policy withholds is removed, with one header note ("Price hidden for your role"); a Stat, a single Value or a Page's Fact shows "Hidden for your role". A record page serves each reader their part of the record: a Fact they may not read is left out of its Facts or Meta, and a Facts, Section or Tab with nothing left they may read is not served (a Split left with one side gives it the whole width) — never a column of lock rows. `audience={["sales"]}` on a Section or Tab places it for those groups only (owners and admins see every one); others are not served it at all (recipe 1). A Preset, facet, Flag, Group or sort that needs a withheld column is dropped for that reader and its count shows as withheld. A derived value, condition, rollup or preview output computed from a withheld column is withheld too. A `when` with a withheld operand is false.
- **Composed values.** A value you compose (a `Fact` whose value is a `Ratio`, a `Ratio` or `Meter` of two fields, a `Text` with `Value`s inside) is never drawn from a part the reader may not read: if any part is withheld the whole composed value is left out quietly (a `Fact` with only that value goes too), never "Hidden for your role" inside it; a derived value or aggregate the read computed for the reader draws, and a part that failed to load still shows its failure (unless another part is withheld: then the whole value leaves). A `Text`, `Badge`, `Callout` or `Empty` with a withheld `Value` inside leaves whole.
- **Rule C — a guard reveals only when it says so.** A condition or guard that reads a column the reader cannot see is unknown to that reader: a verb `offeredWhen` such a condition is offered on no row alike, and a guard over it refuses uniformly, whatever the row holds — unless the model declares that guard `reveals: true`. The page cannot change this, and need not: write the verb; each reader gets the verbs their policy allows.
- **Not run — "Choose …".** A predicate that compares with a ref State or Record that is null, or with a State not chosen yet, does not run its Query (it does not read "every row"): its components show "Choose <noun>", or the first `<Empty when={…}>` that matches. A Section whose `of` is null hides its record title and the components that read it. The [Empty example](components.md#empty) is a page that waits for its patron.
- **Views and records.** A row opens its type's record view — a `RecordPage` of that type (with several, the first by view name; `opens="patron_record"` or `opens="none"` says otherwise) — only when the reader may open it. `audience` on a Page hides the view from other groups in nav and links. A page that is a Form (its Forms run actions) leads to its action: a Link, an `<Action view>` or a nav entry to it is offered only to a reader who may run one of them — a role the action refuses never sees "New patron" lead to "not available for your role". `navLabel` and `icon` compile but are **not drawn yet** (the app chrome, K6.17, draws them from the model); then the nav names a page by its `navLabel` (literal text), else its title when that is text, else the last part of the view name (`Sales.dashboard` → "Dashboard"); a reader who sees two entries with the same name, as an owner sees every group's page, gets each followed by its audience ("Dashboard · Sales"), else its page name — so give pages that share a title a `navLabel` when the audience is not the difference. The nav draws a page with its `icon` (`<Page icon="banknote">`, a Lucide name from lucide.dev/icons; an unknown name is refused with close ones), else its kind's glyph (a record page, a board, a timeline, a form, a table…) — give each nav page its own; an app's nav entry takes no `icon`. A page about a record — a `RecordPage`, or a Page whose title names a record — is named by its title in the app's breadcrumb, the phone's top bar and the browser tab (a reader whose policy withholds the title reads the type's noun, "Patron").

## Compile, read back, fix

`canon views compile canon/views/<view_name>.springml` checks a page against the catalog in the bundle around it (or `--canon <dir>`), offline: every element, prop, name and expression, and the caps. On success it prints the **readback** — the tree, the resting verb of each collection, every scope fill ("`hold_id ← row`") and every input left empty by rule, the read plan with its stages, and the derived chrome (header summary, flagged rows, the queue's "→ Next"). Read it: it is how you see what the reader will get. On failure each refusal names `file:line:col`, a code, the problem and the form to write:

<!-- refused: page.bind.id_field(patron_id) -->
```jsx
<Page title="Loans" lead={loans}>
  <Query id="loans" type="loan" />
  <Section><Table of={loans}><Column field="patron_id" /></Table></Section>
</Page>
```

<!-- refused: cx.name.unknown(filter) -->
```jsx
<Page title="Loans" lead={loans}>
  <Query id="loans" type="loan" />
  <Grid><Stat label="Out" value={loans.filter(l => l.state == "out").length} /></Grid>
</Page>
```

<!-- refused: page.structure.unknown_prop(Table.where) -->
```jsx
<Page title="Loans" lead={loans}>
  <Query id="loans" type="loan" />
  <Section><Table of={loans} where={l => l.state == "out"}><Column field="item" /></Table></Section>
</Page>
```

<!-- refused: page.bind.collection_total(total) -->
```jsx
<Page title="Loans" lead={loans}><Query id="loans" type="loan" /><Grid><Stat label="Loans" value={loans.total} /></Grid></Page>
```

<!-- refused: cx.syntax.unexpected() -->
```jsx
<Page title="Loans" lead={loans}><Query id="loans" type="loan" /><Grid><Stat label="Out" value={loans.count() > 0 ? <Value of={loans.count()} /> : 0} /></Grid></Page>
```

<!-- refused: page.bind.period_initial(span) -->
```jsx
<Page title="Loans" lead={loans}><State id="span" type="range" initial={{from: today(), days: 14}} /><Query id="loans" type="loan" where={l => l.due_on >= span.start} /></Page>
```

<!-- refused: page.structure.unsupported(Aggregate-span) -->
```jsx
<Page title="Loans" lead={loans}><Query id="loans" type="loan" /><Aggregate id="out" type="loan" op="count" by="date" span={["out_on", "due_on"]} /></Page>
```

Other refusals you will meet: `page.bind.lead_many` (two Queries and no `lead`), `page.bind.state_initial` (a date, week, range or month State without `initial`; an object initial is `page.bind.period_initial`), `page.structure.unknown_element` (a name Spring UI does not have), `unknown_field` / `unknown_condition` / `unknown_action` (with "did you mean"), `hops`, `path_key` (a sort, facet, group, lane or total through a link), `id_shadowing` (an id named like a helper: `days`, `today`), `aggregate_arity` (`.at()` with the wrong number of keys), `parent` / `needs_ancestor` (a part outside its parent), `parent_rule` (a Query inside a Section that is not in an Expand), `search_off`, and the verb refusals of [verbs-and-forms.md](verbs-and-forms.md). A refusal that says **"requires PA-n"** names a model feature the catalog cannot declare yet: write the page without that part.

## Caps

Query: page ≤ 200 (unpaged ≤ 500), presets ≤ 8, flags ≤ 3, totals ≤ 16, facets ≤ 3. Aggregate: ≤ 2 keys, ≤ 5,000 cells, `over` ≤ 62 days. Table ≤ 8 columns of importance primary, eyebrow or secondary (mark the rest `detail`); Meta ≤ 6 Facts; ≤ 3 header verbs offered together; ≤ 3 read stages per page.
