<!-- GENERATED from the Spring UI schema and the example pages by `bun run generate:skills` — do not edit -->

# Spring UI components

Spring UI 3.4 has **62 names and no others**: 2 page templates, 5 data elements, 33 components, 22 typed parts. An element the catalog does not name, or a prop an element does not take, is refused at compile time. Each entry says what the name is, where it may sit (**In**), what it may hold (**Holds**), its props, and gives an example page. The language behind them is [language.md](language.md); verbs and forms are [verbs-and-forms.md](verbs-and-forms.md).

**Reading an entry.** **In: any layout** = `Page`, `RecordPage`, `Section`, `Split`, `Columns`, `Tab`, `Form`. **Holds: any component** = every component except `Action` (written where its parent allows it). A prop marked \* is required. Prop types: `text` a literal string · `expr` a render expression in `{…}` · `predicate` a row arrow `{r => …}` (thin narrowing) · `fn` a row function `{r => …}` · `field` a field or link name as a string · `id` the element's name in expressions · `ref:Query` a data element's id as `{q}` · `name:action` (`name:type`, `name:view`, …) a model name as a string · `enum: a / b` one of the listed values · `list:x` an array `{["a", "b"]}` · `object` `{{key: value}}` · `element` JSX such as `<Text>…</Text>`.

**The example model.** Every example compiles against one coach-hire model and omits the `{/* @catalog spring-ui */}` first line each file needs. `trip`: ref, title, destination, depart_date, return_date (dates), passengers, wheelchair_users, vehicle_class, total_cents (money), access_notes, a link `hirer`, and a status machine enquiry → quoted → confirmed → allocated → under_way → completed → closed (or lost, cancelled) run by trip.quote, trip.confirm, trip.allocate, trip.depart, trip.complete, trip.close, trip.mark_lost, trip.cancel. `charge`: trip, trip_ref, trip_date, payer, kind (enum), due_on, amount_cents, paid_cents, open_cents (derived), status (enum) and the condition `is_overdue` (meaning overdue, on due_on, amount open_cents, fix charge.record_payment). `departure`: report_at (timestamp), title, trip_ref, a link `trip`, coach, drivers, first_pickup, group_leader, leader_phone, a machine `state` due → on_road → back (departure.check_out, departure.undo_check_out, departure.check_in) and the condition `is_late` (meaning late, on report_at, fix departure.check_out). `hirer`: name. Actions with page-facing concerns: trip.enquire (steps `trip` and `group`, submits `save` and `quote` whose next is trip.quote), trip.confirm (preview outputs total, passengers, hirer), charge.record_payment (shortLabel "Record", a preCheck guard). For the components that need more: `vehicle` (fleet_no, seats, vehicle_class enum), `class_day` (a derived vehicle_class × date row with booked, available and the condition is_short), `service_slot` (vehicle, starts_at, ends_at, work, date), `workshop_closure` (starts_at, ends_at, reason, date), `driver` (name, licence, week_driving_min), `quote` (trip, version, passengers, total_cents, to-many `lines` of `quote_line`: line_key, description, qty, net_cents); actions service_slot.book, departure.assign_driver (a driver picker with candidates) and trip.add_extras (an array input `extras` whose preview answers each line's net_cents, `preview.rows`). Your page names your model's types, fields, conditions and actions, exactly.

## Index

| Kind | Names |
|---|---|
| Page templates | [Page](#page), [RecordPage](#recordpage) |
| Data elements | [Query](#query), [Record](#record), [Aggregate](#aggregate), [State](#state), [Preview](#preview) |
| Components | [Section](#section), [Split](#split), [Columns](#columns), [Grid](#grid), [Tabs](#tabs), [Toolbar](#toolbar), [Value](#value), [Text](#text), [Badge](#badge), [Callout](#callout), [Stat](#stat), [Ratio](#ratio), [Meter](#meter), [Chart](#chart), [Facts](#facts), [Breakdown](#breakdown), [History](#history), [Lifecycle](#lifecycle), [Table](#table), [List](#list), [Board](#board), [Matrix](#matrix), [Timeline](#timeline), [DayGrid](#daygrid), [Filters](#filters), [Search](#search), [Control](#control), [PeriodNav](#periodnav), [Files](#files), [Compare](#compare), [Action](#action), [Link](#link), [Form](#form) |
| Typed parts | [Preset](#preset), [Flag](#flag), [Tab](#tab), [Series](#series), [Fact](#fact), [Meta](#meta), [Line](#line), [Column](#column), [Item](#item), [Group](#group), [Expand](#expand), [Check](#check), [BulkAction](#bulkaction), [Empty](#empty), [Rows](#rows), [Cell](#cell), [Span](#span), [Field](#field), [Lines](#lines), [Step](#step), [Submit](#submit), [Candidates](#candidates) |

## Page templates

### Page
- A screen. Data elements first, then content; Action children are header verbs. **In:** top level (one per file). **Holds:** `Query`, `Record`, `Aggregate`, `State`, any component, `Action`, `Empty`.
- **Props:** **`title`**\* (text|element) — the page title; text, an expression or a record (its title) · **`subtitle`** (text|element) — the author's line under the title (e.g. the order of the queue) · **`lead`** (ref:Query|list:ref:Query) — the collection(s) that drive the header summary and the nav badge; required when the page has more than one Query; a list sums the attention of each ("2 over · 3 unassigned"). The count is in the type's noun ("23 events"), or, when the lead's `where` (or its active Preset) keeps only rows holding a named condition that names a count (both `label` and `labelOne`), in that noun (`where={i => i.is_overdue}` with label "overdue instalments" → "14 overdue instalments", `labelOne` for 1); a condition with a `label` alone ("Live") is a tag and leaves the type's noun; a lead whose `where` holds two such conditions is refused (`lead_noun_ambiguous`): declare one condition that is both, with the noun the count reads in; a lead whose `where` keeps rows of a condition that names no count compiles with a warning (`lead_noun_missing`) — declare its `label` and `labelOne` ("overdue instalments" / "overdue instalment"), or the header counts those rows in the type's noun as if they were all of them · **`params`** (object) — the view's parameters and their types: {event: "event"}; param() reads them · **`audience`** (list:group) — groups that may open this view (checked against the org's groups); others do not see it in nav or links (reads stay policy-checked) · **`navLabel`** (text) — the page's name in the app's nav, breadcrumb and Home (literal text); default: the title when it is text, else the view name's last part (`Sales.dashboard` → "Dashboard"). A reader who sees two nav entries with the same name sees each followed by its audience ("Dashboard · Sales"), else its page name · **`icon`** (text) — the page's glyph in the app's nav (literal text): a kebab-case Lucide name from the platform's pinned set (lucide.dev/icons), e.g. "banknote". Default, by what the page is: a RecordPage "file-text"; else the first the page holds of Board "kanban", Timeline "chart-gantt", DayGrid "calendar-days", Matrix "calendar-range", Form "square-pen", Compare "file-diff", Files "paperclip", Chart "chart-column", Table or List "list"; else "layout-dashboard". Give each nav page its own
```jsx
<Page title="Payments" subtitle="Oldest open charge first" lead={pay}>
  <Query id="pay" type="charge" sort="due_on"><Flag condition="is_overdue" /></Query>
  <Action run="charge.record_payment" role="primary" />
  <Section><Table of={pay}><Column field="trip_ref" importance="primary" /><Column field="open_cents" total="sum" /></Table></Section>
</Page>
```

### RecordPage
- A record's screen. Reads param("id"); the record is in scope as `record`; Action children are record header verbs. **In:** top level (one per file). **Holds:** `Query`, `Record`, `Aggregate`, `State`, `Meta`, any component, `Action`, `Empty`.
- **Props:** **`type`**\* (name:type) — the record's type · **`subtitle`** (text|element) · **`lead`** (ref:Query) · **`audience`** (list:group) · **`icon`** (text) — as on `Page`
```jsx
<RecordPage type="trip">
  <Meta><Fact field="ref" /><Fact field="depart_date" /><Fact field="passengers" /></Meta>
  <Action run="trip.confirm" role="primary" />
  <Section title="Details"><Facts><Fact field="hirer" /><Fact label="Away" span={["depart_date", "return_date"]} /></Facts></Section>
</RecordPage>
```

## Data elements

### Query
- The collection: a declared slice of one type plus the reader's working state (preset, facets, search, sort, page, selection). Served by one server batch. **In:** `Page`, `RecordPage`, `Section`; in `Section` only inside `Expand`. **Holds:** `Preset`, `Flag`.
- **Props:** **`id`**\* (id) — the name in expressions; may not shadow a helper or scope name · **`type`**\* (name:type) · **`where`** (predicate) — thin narrowing (see language.md, predicates) · **`sort`** (field|list:field) — "due_on", "-version" or a list; own or 1-hop fields; a link sorts by its target's title; an enum by its declared values; a named condition sorts true first with "-" ("-is_full"); blanks always last · **`page`** (number, default 25) — rows per page (≤ 200); 0 = unpaged (≤ 500). A Query a Timeline, DayGrid, Span or Matrix draws takes no page: those have no pager and read it whole over what they show (≤ 500 rows, then "Showing 500 of N") · **`search`** (list:field|boolean) — own or 1-hop text fields; default the type's searchable; false turns Search off · **`facets`** (list:field) — ≤ 3 enum, set<enum> or link fields shown as facet chips · **`current`** (predicate) — the first row in sort order that matches and carries no critical condition is the current ("→ Next") row · **`nameBy`** (field) — the field that names a row in the queue header ("late: 07:15 SR-14"), e.g. the vehicle; default the title, or the eyebrow when the title is longer than 24 characters
```jsx
<Page title="Payments" lead={pay}>
  <Query id="pay" type="charge" sort="due_on" page={14} facets={["kind"]}>
    <Preset id="open" label="Open" where={c => ["open", "part_paid"].includes(c.status)} default />
    <Preset id="all" label="All" />
    <Flag condition="is_overdue" />
  </Query>
  <Toolbar><Filters of={pay} /><Search of={pay} /></Toolbar>
  <Section><Table of={pay}><Column field="trip_ref" importance="primary" /><Column field="due_on" /><Column field="open_cents" total="sum" /></Table></Section>
</Page>
```

### Record
- One record by key. **In:** `Page`, `RecordPage`. **Holds:** nothing.
- **Props:** **`id`**\* (id) · **`type`**\* (name:type) · **`key`**\* (expr) — the record's key: param("trip"), another Record's link, or a date for a per-day type whose ids are dates; a type with prefixed ids ("day_…") is read by its date field with a Query and a State auto="single" (a date key is refused)
```jsx
<Page title={hr} params={{hirer: "hirer"}}>
  <Record id="hr" type="hirer" key={param("hirer")} />
  <Query id="hTrips" type="trip" sort="-depart_date" where={t => t.hirer == hr} />
  <Section title="Trips"><Table of={hTrips}><Column field="title" importance="primary" /><Column field="depart_date" /><Column field="status" /></Table></Section>
</Page>
```

### Aggregate
- One grouped read, keyed; read with `.at()`. **In:** `Page`, `RecordPage`. **Holds:** nothing.
- **Props:** **`id`**\* (id) · **`type`** (name:type) — the type to aggregate; or use `of` · **`of`** (ref:Query) — inherit the Query's type and the rows it reads: its base where (incl. bound States) and the window the components drawing it show (a Matrix's columns, a Timeline's or DayGrid's range, a State's weekdays), never its page, preset or search; may only narrow — so a Stat of it agrees with the Matrix, Timeline or DayGrid that draws the Query · **`op`**\* (enum: count / sum / min / max / avg / list) · **`property`** (field) — the measured field (sum/min/max/avg); for list: the link whose target records are listed — without it the aggregated rows themselves are listed (they are the popover records) · **`by`** (field|list:field) — ≤ 2 keys: own or 1-hop fields; date keys may end in @day, @week, @month, @bucket; with span the day key is written `date` · **`on`** (field) — the date or timestamp that places a row on a day (default: the single date key) · **`span`** (list:field) — [start, end]: the row counts on every day it covers · **`over`** (expr) — a day series: days(week) · **`window`** (object) — {last: n} buckets for @week/@month keys · **`where`** (predicate) · **`facts`** (list:field) — with op=list: ≤ 3 fields of each listed record (the rows, or the `property` targets), shown in the popover · **`decimals`** (number) — decimal places its numbers are written to (0–6), the word a number property's `scale: { decimals }` uses; default: the measure's declared decimals — an `avg` of a field that declares none to 1 place
```jsx
<Page title="Today" lead={deps}>
  <Query id="deps" type="departure" sort="report_at" page={50} />
  <Aggregate id="byState" type="departure" op="count" by="state" />
  <Aggregate id="seats" type="trip" op="sum" property="passengers" where={t => t.status == "confirmed"} />
  <Grid>
    <Stat label="On the road" value={byState.at("on_road")} />
    <Stat label="Passengers confirmed" value={seats.value} />
  </Grid>
</Page>
```

### State
- A value the reader chooses; kept in the URL under its id. **In:** `Page`, `RecordPage`. **Holds:** nothing.
- **Props:** **`id`**\* (id) · **`type`**\* (enum|name:type: date / week / range / month / boolean / number / string / enum) — or an ObjectType name = a ref State · **`initial`** (expr) — required for date/week/range/month; a range's is {from: <date>, days: <whole number>}, e.g. {{from: today(), days: 14}} (never {start, end}); wins over auto when not null · **`options`** (list:object) — enum States: [{value, label}] · **`where`** (predicate) — ref States without `from`: the choices · **`sort`** (field|list:field) — ref States without `from`; a link sorts by its target's title, as a Query's · **`from`** (ref:Query) — ref States: the choices are this Query's rows — its where, active preset, chosen facets and search, in its sort, never just its page — so the picker lists what the page lists; no where or sort of its own · **`auto`** (enum: single / first / second / current) — ref States: pick the only choice, the first/second in sort, or the `from` Query's current row. Without auto nothing is picked until the reader chooses (a filter State reads null as "all"): `from={upcoming} auto="first"` opens on the first upcoming row · **`weekdays`** (list:string) — week/range States: the days shown, e.g. ["mon",…,"sat"]; default every day. days(week), a Matrix, a Timeline day axis, a week DayGrid and PeriodNav (its label and day strip) show only these
```jsx
<Page title="Trips this week" lead={wk}>
  <State id="week" type="week" initial={weekOf(today())} weekdays={["mon", "tue", "wed", "thu", "fri", "sat"]} />
  <Query id="wk" type="trip" sort="depart_date" page={0} where={t => t.depart_date <= week.end && t.return_date >= week.start} />
  <Toolbar><PeriodNav state={week} span="week" /></Toolbar>
  <Section><Table of={wk}><Column field="title" importance="primary" /><Column field="depart_date" /></Table></Section>
</Page>
```

### Preview
- The Form action's live preview outputs as `pv.<output>` (typed, policy-checked). **In:** `Form` (needs a `Form` ancestor). **Holds:** nothing.
- **Props:** **`id`**\* (id)
```jsx
<RecordPage type="trip">
  <Form run="trip.confirm">
    <Preview id="pv" />
    <Grid><Stat label="Total" value={pv.total} /><Stat label="Passengers" value={pv.passengers} /></Grid>
    <Submit />
  </Form>
</RecordPage>
```

## Components

### Section
- A titled region; a top-level Section is one card with its title in the header. **In:** `Page`, `RecordPage`, `Section`, `Split`, `Columns`, `Tab`, `Expand`, `Form`, `Step`. **Holds:** any component, `Action`, `Empty`, `Query`, `Field`, `Lines`, `Step`, `Submit`, `Candidates`.
- **Props:** **`title`** (text|element|expr) — text, or a record expression (its title) · **`of`** (expr) — the section's record: scope for verbs (incl. its Toolbar), default record of Facts; when null, a record title hides, components that read it render nothing and the Section's Empty speaks · **`when`** (expr) — render expression; false hides the element (UX only, never security). A withheld operand makes it false. · **`audience`** (list:group) — groups it is drawn for (checked against the org's groups); a reader in none of them is not served it at all — no title, no lock rows (owners and admins see every one); decided on the server, not a data boundary (reads stay policy-checked)
```jsx
<RecordPage type="trip">
  <Query id="tch" type="charge" sort="due_on" where={c => c.trip == record} />
  <Section title="Charges" of={record}>
    <Action run="trip.quote" />
    <Table of={tch}><Column field="kind" importance="primary" /><Column field="amount_cents" total="sum" /></Table>
  </Section>
</RecordPage>
```

### Split
- Main + aside, exactly two children; the aside is sticky at ≥ 1024 px when it holds a Form summary. In a Form, the side holding its inputs (Field, Lines, Step, Submit, Candidates) is sized by its fields and the result side takes the rest, in either order. **In:** any layout. **Holds:** any component, `Field`, `Lines`, `Step`, `Submit`, `Candidates`.
- **Props:** none.
- **Example:** recipe 5 (`Trip.record`) in [recipes.md](recipes.md).

### Columns
- 2–4 different children side by side when the container is ≥ 960 px; stacked otherwise. **In:** any layout. **Holds:** any component.
- **Props:** none.
- **Example:** recipe 6 (`Trip.week`) in [recipes.md](recipes.md).

### Grid
- Like items only: Stat or Files tiles; the tile count picks the tracks at every width — wide, 2–5 tiles one track each (3 tiles → 3, never 3 of 4), 6 → 3, 7 or more → 4; on phone 2 (3 for 3, 1 for 5 or 7) — so the first row has no empty track and no later row is an orphan; tiles never stretch; a withheld tile reflows the rest left. **In:** any layout. **Holds:** `Stat`, `Files`.
- **Props:** none.
- **Example:** the [Aggregate](#aggregate) example.

### Tabs
- A tool's views (a builder, an editor), or record sections you group yourself; a long RecordPage draws its sections as tabs on its own. **In:** any layout. **Holds:** `Tab`.
- **Props:** none.
```jsx
<RecordPage type="trip">
  <Query id="tch" type="charge" sort="due_on" where={c => c.trip == record} />
  <Tabs>
    <Tab label="Details"><Facts><Fact field="destination" /><Fact field="passengers" /></Facts></Tab>
    <Tab label="Charges" lead={tch}><Table of={tch}><Column field="kind" importance="primary" /><Column field="amount_cents" /></Table></Tab>
  </Tabs>
</RecordPage>
```

### Toolbar
- Inputs and verbs for the page or a section (a Section's first child). **In:** any layout. **Holds:** `Search`, `Filters`, `Control`, `PeriodNav`, `Action`.
- **Props:** none.
- **Example:** the [Query](#query) example.

### Value
- One field or typed expression, shown the one way its type and container show it. **In:** `Page`, `RecordPage`, `Section`, `Split`, `Columns`, `Tab`, `Text`, `Badge`, `Callout`, `Empty`, `Cell`, `Form`, `Lines`. **Holds:** nothing.
- **Props:** **`of`**\* (expr) — a field, link, aggregate cell, list, handle or typed expression · **`change`** (boolean, default false) — this is a difference: show the sign (+15) and the `better` tone
```jsx
<Page title="Payments" lead={pay}>
  <Query id="pay" type="charge" sort="due_on"><Flag condition="is_overdue" /></Query>
  <Section title="Overdue">
    <Text><Value of={pay.count("is_overdue")} /> charges are overdue, <Value of={pay.amount("is_overdue")} /> still open.</Text>
  </Section>
</Page>
```

### Text
- Prose with inline values. **In:** `Page`, `RecordPage`, `Section`, `Split`, `Columns`, `Tab`, `Cell`, `Form`. **Holds:** text, `Value`, `Link`, `Badge`.
- **Props:** **`when`** (expr) — as on `Section`
```jsx
<Page title="Today" subtitle={<Text>Departure board · now <Value of={now()} /></Text>} lead={deps}>
  <Query id="deps" type="departure" sort="report_at" />
  <Section><List of={deps}>{d => <Item lead={d.report_at} title={d.title} />}</List></Section>
</Page>
```

### Badge
- A neutral mark with text ("Rush", "Repeat"); attention marks come from Flags, never from a Badge. **In:** `Page`, `RecordPage`, `Section`, `Split`, `Columns`, `Tab`, `Text`, `Callout`, `Item`, `Form`. **Holds:** text, `Value`.
- **Props:** **`when`** (expr) — as on `Section`
```jsx
<Page title="Trips" lead={trips}>
  <Query id="trips" type="trip" sort="depart_date" />
  <Section><List of={trips}>{t => <Item title={t.title} sub={[t.destination]}><Badge when={t.wheelchair_users > 0}>Wheelchair</Badge></Item>}</List></Section>
</Page>
```

### Callout
- A note the author writes: info, or a warning about the screen's content (not a refusal, not a data exception). **In:** any layout. **Holds:** text, `Value`, `Link`, `Badge`.
- **Props:** **`tone`** (enum: info / warn, default info) · **`title`** (text) · **`when`** (expr) — as on `Section`
```jsx
<Page title="Quotes" lead={q}>
  <Query id="q" type="trip" where={t => t.status == "quoted"} />
  <Callout tone="warn" title="Quotes expire after 14 days">Chase the hirer before the date in the last column.</Callout>
  <Section><Table of={q}><Column field="title" importance="primary" /><Column field="depart_date" /></Table></Section>
</Page>
```

### Stat
- A KPI tile: label, value, optional delta, foot line and sparkline. **In:** `Page`, `RecordPage`, `Section`, `Split`, `Columns`, `Grid`, `Tab`, `Form`. **Holds:** `Ratio`.
- **Props:** **`label`**\* (text) · **`value`** (expr) — number, money or duration; or a Ratio child · **`compare`** (enum: previous_period / same_period_last_year) — shifts the time key of the Aggregate in value; a period in progress is set against the same elapsed span of the earlier one ("on 1–10 Nov"); for the whole earlier period, show its total in a Ratio child instead · **`better`** (enum: up / down) — default from the field's `better` · **`context`** (element) — the foot line · **`trend`** (ref:Aggregate) — keyed by time → sparkline · **`decimals`** (number) — decimal places a number value is written to (0–6), the word a number property's `scale: { decimals }` uses; default: its field's or Aggregate's decimals; a computed figure (`a.value * 100 / b.value`) that is not whole to 1 place, and arithmetic keeps the unit of its typed side · **`when`** (expr) — as on `Section`
- **Example:** recipe 6 (`Trip.week`) in [recipes.md](recipes.md).

### Ratio
- "assigned / needed": the numerator in 600 ink, "/ needed" muted. Its tone comes from its two numbers, never from a prop: `of` is a whole (capacity), `need` a requirement. **In:** `Page`, `RecordPage`, `Section`, `Split`, `Columns`, `Tab`, `Stat`, `Cell`, `Form`. **Holds:** nothing.
- **Props:** **`value`**\* (expr) · **`of`** (expr) — the whole the value is a share of (capacity, available, total): over it is critical "over by n"; at or under it the pair is neutral; a literal (`of={2400}`) is in the value's unit (minutes: "40 h") · **`need`** (expr) — what is needed (staff per shift, coaches per trip): below it is short — critical "short n"; at it met — a check; above it over — warn "over by n"; a literal is in the value's unit · **`remainder`** (text) — with `need`: the words for what is still missing below it, after its count — `remainder="to check"` reads "26 / 35 · 9 to check" where the default reads "short 9" (lines not yet checked are not short); literal text
- **Example:** recipe 1 (`Departure.today`) in [recipes.md](recipes.md).

### Meter
- A bar with its number; tone from the field's `thresholds` (model), healthy = neutral ink. **In:** any layout. **Holds:** nothing.
- **Props:** **`value`**\* (expr) · **`of`**\* (expr)
- **Example:** recipe 5 (`Trip.record`) in [recipes.md](recipes.md).

### Chart
- A chart by question; the platform picks bars/lines/stacking. **In:** any layout. **Holds:** `Series`.
- **Props:** **`shows`**\* (enum: trend / composition / comparison / capacity) · **`limit`** (expr) — capacity line · **`when`** (expr) — as on `Section`
- **Example:** recipe 6 (`Trip.week`) in [recipes.md](recipes.md).

### Facts
- Key–value facts of a record. **In:** any layout. **Holds:** `Fact`.
- **Props:** **`of`** (expr) — the record (default: the nearest Section's `of`, else `record`); a link (record.space) carries the target's title only, so its Facts are refused: read it with a Record and name that
- **Example:** the [RecordPage](#recordpage) example.

### Breakdown
- A money summary; lines in written order (it shows, it does not add). **In:** any layout. **Holds:** `Line`.
- **Props:** **`of`**\* (expr) — a record or a Preview
- **Example:** recipe 5 (`Trip.record`) in [recipes.md](recipes.md).

### History
- A record's changes: its transitions and its history: true fields (PA-8), or a declared event log. **In:** any layout. **Holds:** nothing.
- **Props:** **`of`**\* (expr) · **`fields`** (list:field) — the type's opt-in history fields · **`log`** (field) — a to-many link of an event type to read instead of the transition log · **`at`** (field, default at) — the event's date or timestamp · **`by`** (field, default by) — who: a field or a link (its record's title) · **`action`** (field, default action) · **`note`** (field, default note)
```jsx
<RecordPage type="trip">
  <Lifecycle of={record} />
  <Split>
    <Section title="What happened"><History of={record} /></Section>
    <Section title="Documents"><Files of={record} /></Section>
  </Split>
</RecordPage>
```

### Lifecycle
- Where a record is in its process: the machine's main path with dates; a taken branch replaces the rest of the path. **In:** any layout. **Holds:** nothing.
- **Props:** **`of`**\* (expr)
- **Example:** the [History](#history) example.

### Table
- Aligned columns over a collection. **In:** any layout. **Holds:** `Column`, `Action`, `BulkAction`, `Group`, `Expand`, `Check`, `Empty`.
- **Props:** **`of`**\* (ref:Query|ref:Aggregate|expr) — a Query, an Aggregate (one row per key) or a Preview array (pv.items) · **`opens`** (expr|string, default auto) — "auto" (the type's record view; rows of a type without one, Aggregate rows and preview rows do not open), a view name, a record expression (opens its view) or "none" · **`picks`** (ref:State) — a ref State of the row type: a row click sets it (master–detail); not drawn yet, refused at compile: choose the record with a Control on the ref State, or let the rows open their record (opens) · **`verbs`** (enum: flag / job / rest / menu, default flag) — flag (default): a verb is visible only on rows whose flag it fixes (the condition's `fix`), as a bordered button; job: the first offered verb is the page's per-row job — a quiet text button (the action's `shortLabel`: "Record") on every eligible row, bordered on flagged rows; `rest` is a synonym of job; menu: every verb in ⋯. A queue (a Query with `current`, or a Check) defaults to job. A verb whose preCheck guard fails never shows at rest. ⋯ appears only when it holds a verb (opening the record is the title's job, not a menu item); a job column may show different verbs per row (the first offered differs by state); a create with no subject may be a row verb, its inputs filled from the row. · **`when`** (expr) — as on `Section`
- **Example:** recipe 3 (`Trip.list`) in [recipes.md](recipes.md).

### List
- Free rows with the card anatomy (queues, run sheets). **In:** any layout. **Holds:** a child function returning `Item`, `Action`, `Group`, `Check`, `Empty`.
- **Props:** **`of`**\* (ref:Query|expr) — a Query or a Preview array · **`opens`** (expr|string, default auto) — as on `Table` · **`picks`** (ref:State) — not drawn yet, refused at compile: choose the record with a Control on the ref State, or let the rows open their record (opens) · **`verbs`** (enum: flag / job / rest / menu, default flag) — as on `Table` · **`when`** (expr) — as on `Section`
- **Example:** recipe 1 (`Departure.today`) in [recipes.md](recipes.md).

### Board
- Cards in lanes of a process. **In:** any layout. **Holds:** a child function returning `Item`, `Action`, `Empty`.
- **Props:** **`of`**\* (ref:Query) — the collection · **`opens`** (expr|string, default auto) — as on `Table` · **`by`**\* (field) — an enum or state-machine field · **`sum`** (field) — money/number (incl. rollups) summed in each lane header · **`when`** (expr) — as on `Section`
- **Example:** recipe 4 (`Trip.board`) in [recipes.md](recipes.md).

### Matrix
- Rows × columns of cells. **In:** any layout. **Holds:** `Rows`, `Empty`.
- **Props:** **`cols`**\* (expr) — the grain: days(week) (honours the State's weekdays), weeks(range) or months(range), hours(day), or an enum field "type.field" · **`cells`** (ref:Query) — default for every Rows: a Query of a per-cell type (e.g. class_day) whose Flags give each cell its attention · **`cellKey`** (list:field) — the cells type's fields matching (row, column), e.g. ["vehicle_class", "date"]; ["date"] for a fixed row · **`closed`** (ref:Query) — the closed days (a plant's holidays and Sundays): each row closes the day column its `closedKey` date falls on, read whole over the columns' days, so its where needs no window term · **`closedKey`** (field) — the closed Query's date field, e.g. "date" (required with `closed`) · **`closedReason`** (field) — the closed Query's field naming why, e.g. "holiday_name" (default: the row's title) · **`corner`** (text) · **`opens`** (expr|string, default auto) — the row label's target (auto: the row record's view when rows are records) · **`when`** (expr) — as on `Section`
```jsx
<Page title="Coaches by class" lead={cd}>
  <State id="week" type="week" initial={weekOf(today())} />
  <Query id="cd" type="class_day" page={0} where={x => x.date >= week.start && x.date <= week.end}><Flag condition="is_short" /></Query>
  <Aggregate id="going" type="trip" op="list" by={["vehicle_class", "date"]} span={["depart_date", "return_date"]} over={days(week)} />
  <Toolbar><PeriodNav state={week} span="week" /></Toolbar>
  <Section title="Booked / available">
    <Matrix cols={days(week)} cells={cd} cellKey={["vehicle_class", "date"]} corner="Class">
      <Rows values="vehicle.vehicle_class">{(c, d, x) => <Cell list={going.at(c, d)}><Ratio value={x.booked} of={x.available} /></Cell>}</Rows>
      <Empty title="No coaches">Add vehicles to plan the week.</Empty>
    </Matrix>
  </Section>
</Page>
```

### Timeline
- Items over time in lanes. **In:** any layout. **Holds:** a child function returning `Item`, `Span`, `Action`, `Empty`.
- **Props:** **`of`**\* (ref:Query) · **`lane`**\* (field) — a link or enum field; null lanes collect in a derived lane · **`lanes`** (ref:Query) — a link lane's lane records: a Query of the link's target type, every row a lane in its order, an empty lane kept (the whole fleet, idle coaches too), and no other: an item on a lane it leaves out is not drawn. Omitted: only the lanes that hold an item in the range, in title order. An enum lane draws every value · **`start`**\* (field|list:field) — a timestamp or date, or [date, time] · **`end`**\* (field|list:field) — blank on a row = open-ended: drawn to the axis end with an open-end mark; never coalesce it to a far date. The Timeline reads its Query over the range itself, so the Query needs no window term: one on the end field (`c.end_date >= span.start`) drops every open-ended item, since a blank never matches `>=` · **`range`**\* (ref:State|expr) — a week, range or month State, or a row-free expression; a date State is an hour axis · **`grain`** (enum: day / week / month, default day) — the axis columns: days, the weeks (from the org's week start) or the months the range touches; not with a date State · **`capacity`** (expr|string, default 1) — how much one lane holds at once; a bar is marked Conflict only while its lane holds more. 1 (the default: a coach, a room or a person does one thing at a time) marks any two bars at once; a number or a row-free expression (`capacity={record.owned_qty}`) is every lane's; a lane function over the `lanes` records (`capacity={k => k.owned_qty}`) is each lane's own; with a second parameter, the day, it is a capacity per day (`capacity={(k, d) => k.owned_qty + hired.at(k, d)}`: owned plus that day's hire-ins, hired a sum Aggregate keyed by lane and date), measured at every day start as well as every bar start; "none" marks no overlap (a lane of notes, a day of events). A lane whose capacity reads blank or withheld marks no overlap · **`amount`** (field) — a number field of the item: how much of its lane's capacity each bar takes (`amount="qty"`: 30 + 20 tables of 45 owned is over, 30 + 10 is not); without it each bar takes 1; a blank or withheld amount takes 0. Needs a capacity other than "none". Kit hire: `<Timeline of={res} lanes={kit} lane="kit_item" start="out_on" end="back_on" range={span} capacity={k => k.owned_qty} amount="qty">` · **`opens`** (expr|string, default auto) · **`move`** (name:action) — not drawn yet, refused at compile: put <Action run=…> inside the Timeline, each bar offers it with the bar as its subject · **`when`** (expr) — as on `Section`
```jsx
<Page title="Contracts by coach" lead={cons}>
  <State id="span" type="range" initial={{from: addDays(today(), -14), days: 84}} />
  <Query id="cons" type="contract" sort="start_date" />
  <Query id="fleet" type="vehicle" sort="fleet_no" />
  <Query id="shut" type="workshop_closure" />
  <Toolbar><PeriodNav state={span} span="month" /></Toolbar>
  <Section title="Contracts">
    <Timeline of={cons} lanes={fleet} lane="vehicle" start="start_date" end="end_date" range={span} grain="week">
      {c => <Item title={c.number} />}
      <Span of={shut} kind="blocked" start="starts_at" end="ends_at" label="reason" />
    </Timeline>
  </Section>
</Page>
```

### DayGrid
- The calendar: lanes × time of day. **In:** any layout. **Holds:** a child function returning `Item`, `Span`, `Action`, `Empty`.
- **Props:** **`of`**\* (ref:Query) · **`lanes`** (ref:Query) — resources; may be a per-day type; omitted with a week range = one column per day · **`laneKey`** (field) — the lanes row's link to the lane record (when lanes is a per-day type) · **`lane`** (field) — the item's link to the lane record; required with `lanes` · **`start`**\* (field) · **`end`**\* (field) · **`range`**\* (ref:State|expr) · **`hours`** (object) — {from: "08:00", to: "19:00"} · **`slot`** (enum: 15 / 30 / 60, default 15) · **`create`** (name:action) — not drawn yet, refused at compile: put <Action run=…> on the page or its Section, its dialog asks for the start and the lane · **`move`** (name:action) — not drawn yet, refused at compile: put <Action run=…> inside the DayGrid, each block offers it with the block as its subject · **`opens`** (expr|string, default auto) · **`when`** (expr) — as on `Section`
```jsx
<Page title="Workshop" lead={slots}>
  <State id="day" type="date" initial={today()} />
  <Query id="bays" type="vehicle" sort="fleet_no" page={0} />
  <Query id="slots" type="service_slot" page={0} where={s => s.date == day} />
  <Query id="shut" type="workshop_closure" page={0} where={c => c.date == day} />
  <Toolbar><PeriodNav state={day} span="day" /></Toolbar>
  <Section title="Service bays">
    <DayGrid of={slots} lanes={bays} lane="vehicle" start="starts_at" end="ends_at" range={day} hours={{from: "07:00", to: "18:00"}} slot="30">
      {s => <Item title={s.work} />}
      <Span of={shut} kind="blocked" start="starts_at" end="ends_at" label="reason" />
    </DayGrid>
  </Section>
</Page>
```

### Filters
- Preset chips and facet chips with live counts. **In:** `Page`, `RecordPage`, `Section`, `Split`, `Columns`, `Tab`, `Toolbar`, `Form`. **Holds:** nothing.
- **Props:** **`of`**\* (ref:Query)
- **Example:** the [Query](#query) example.

### Search
- Free text over the Query's search fields ($search). **In:** `Page`, `RecordPage`, `Section`, `Split`, `Columns`, `Tab`, `Toolbar`, `Form`. **Holds:** nothing.
- **Props:** **`of`**\* (ref:Query)
- **Example:** the [Query](#query) example.

### Control
- One control for one State; the kind follows the State's type. **In:** `Page`, `RecordPage`, `Section`, `Split`, `Columns`, `Tab`, `Toolbar`, `Form`. **Holds:** nothing.
- **Props:** **`state`**\* (ref:State) · **`label`**\* (text)
- **Example:** the [Empty](#empty) example.

### PeriodNav
- ‹ Today › with the range label; drives every component bound to its State. **In:** `Page`, `RecordPage`, `Section`, `Split`, `Columns`, `Tab`, `Toolbar`, `Form`. **Holds:** nothing.
- **Props:** **`state`**\* (ref:State) · **`span`**\* (enum: day / week / fortnight / month) — the step: a date State steps by day, a week by week, a month by month, a range by its length (fortnight) or by a week or a month (its length kept; a range of whole months keeps its months) · **`counts`** (ref:Aggregate|list:ref:Aggregate) — keyed by date: a day strip with badges in the header's unit; a list is summed (coach-short and driver-short days) · **`skip`** (enum: none / empty, default none) — empty: arrows step to the next day with a count
- **Example:** the [State](#state) example.

### Files
- A record's attachments: tiles, lightbox, Add when the reader may attach. **In:** `Page`, `RecordPage`, `Section`, `Split`, `Columns`, `Grid`, `Tab`, `Form`. **Holds:** nothing.
- **Props:** **`of`**\* (expr) · **`kinds`** (list:string) — attachment kinds of the type (incl. kinds backed by a child type, PA-14)
- **Example:** the [History](#history) example.

### Compare
- Two versions of a record with aligned lines, signed deltas and totals. **In:** any layout. **Holds:** nothing.
- **Props:** **`a`**\* (expr) · **`b`**\* (expr) · **`lines`**\* (field) — a to-many link · **`key`**\* (field) — unique per record: a field or a link (the line's item); a duplicate is a render refusal naming it · **`value`**\* (field) — the line money field: per-line signed delta, totals per side, difference · **`fields`** (list:field) — line fields compared and shown; a line is Changed when any differs · **`facts`** (list:field) — header fields aligned above the lines, each with its change · **`total`** (field) — the record's own money or number total (e.g. incl. VAT): a/b and the difference
```jsx
<Page title="Compare quotes" params={{trip: "trip"}}>
  <Record id="tr" type="trip" key={param("trip")} />
  <State id="va" type="quote" where={q => q.trip == tr} sort="-version" auto="second" />
  <State id="vb" type="quote" where={q => q.trip == tr} sort="-version" auto="first" />
  <Toolbar><Control state={va} label="Version" /><Control state={vb} label="Against" /></Toolbar>
  <Section title="Lines">
    <Compare a={va} b={vb} lines="lines" key="line_key" value="net_cents" total="total_cents" fields={["qty", "net_cents"]} facts={["passengers"]} />
  </Section>
</Page>
```

### Action
- The one verb element: runs an action (generated dialog) or opens a view. **In:** `Page`, `RecordPage`, `Section`, `Tab`, `Toolbar`, `Table`, `List`, `Group`, `Empty`, `Board`, `Cell`, `Timeline`, `DayGrid`. **Holds:** nothing.
- **Props:** **`run`** (name:action) — an action · **`view`** (name:view) — a view to open · **`params`** (object|fn) — with view: named parameters, or a row function (r => ({customer: r.customer})) so a Form page can be a row's fix verb · **`label`** (text) — required with view; refused with run (the model's label) · **`role`** (enum: primary) — may sit on verbs that exclude each other by legality: the first offered one is primary · **`with`** (object|fn) — input presets: {{event_id: evt}} or a row function n => ({position: n.position}); null clears a fill · **`rest`** (boolean) — false: never the at-rest row verb · **`when`** (expr|fn) — render expression; false hides the element (UX only, never security). A withheld operand makes it false. On a Table/List/Board row verb, a row function d => d.state == "on_road" is its row condition: the verb is offered only on the rows it holds for.
```jsx
<RecordPage type="trip">
  <Action run="trip.cancel" />
  <Action run="trip.quote" />
  <Action run="trip.confirm" role="primary" />
  <Action view="Trip.list" label="All trips" />
</RecordPage>
```

### Link
- Inline navigation to a view or a record. **In:** `Page`, `RecordPage`, `Section`, `Split`, `Columns`, `Tab`, `Text`, `Callout`, `Empty`, `Form`. **Holds:** text.
- **Props:** **`view`** (name:view) · **`params`** (object) · **`record`** (expr) — opens the record's view
```jsx
<RecordPage type="trip">
  <Section title="Hirer"><Text>Booked by <Link record={record.hirer}>the hirer</Link> · <Link view="Trip.list">all trips</Link></Text></Section>
</RecordPage>
```

### Form
- An action's form as the screen; a scope: any layout inside; Field/Lines/Step/Submit/Candidates/Preview need a Form ancestor. **In:** any layout. **Holds:** `Preview`, any component, `Field`, `Lines`, `Step`, `Submit`, `Candidates`, `Empty`.
- **Props:** **`run`**\* (name:action) · **`with`** (object) — input presets, e.g. {{proposal_id: param("proposal")}}; a null value leaves the input to its model default · **`view`** (name:view) — the view a successful submit opens, with `params` (built like a Link: a view the reader may not open, or a param with no value, opens nothing). Without it the Form resets in place, or opens the record the action navigates to (`presentation.refresh: "navigate"`) · **`params`** (object) — with view: its named parameters, from the page's scope ({{event: ev}})
```jsx
<Page title="New enquiry">
  <Form run="trip.enquire">
    <Step name="trip"><Field name="hirer_id" /><Field name="title" /><Field name="destination" /><Field name="depart_date" /><Field name="return_date" /></Step>
    <Step name="group"><Field name="passengers" /><Field name="wheelchair_users" /><Field name="access_notes" /><Field name="vehicle_class" /></Step>
    <Submit mode="save" />
    <Submit mode="quote" />
  </Form>
</Page>
```

## Typed parts

### Preset
- A saved view of the collection; Filters shows it as a chip with its live count. **In:** `Query`. **Holds:** nothing.
- **Props:** **`id`**\* (id) · **`label`**\* (text) · **`where`** (predicate) — no where = all rows · **`default`** (boolean, default false) — at most one; none = the first Preset · **`short`** (text) — the chip label at phone width ("Due 14 d"); a chip never clips its label · **`verbs`** (enum: flag / job / menu) — the collection's verbs mode while this preset is active ("Ready to issue" → job, "Overdue" → flag)
- **Example:** the [Query](#query) example.

### Flag
- Marks rows that need attention, from a named condition of the row type. **In:** `Query`. **Holds:** nothing.
- **Props:** **`condition`**\* (name:condition) — a named condition (Part A PA-3); meaning, level, on, amount and label come from the model
- **Example:** the [Query](#query) example.

### Tab
- One tab. **In:** `Tabs`. **Holds:** any component, `Action`, `Empty`, `Field`, `Lines`, `Step`, `Submit`, `Candidates`.
- **Props:** **`label`**\* (text) · **`lead`** (ref:Query) — the collection whose total is the tab's count badge (default: the tab's first collection) · **`audience`** (list:group) — as on `Section`
- **Example:** the [Tabs](#tabs) example.

### Series
- One series of a Chart. **In:** `Chart`. **Holds:** nothing.
- **Props:** **`of`**\* (ref:Aggregate) · **`label`**\* (text)
- **Example:** recipe 6 (`Trip.week`) in [recipes.md](recipes.md).

### Fact
- One fact. **In:** `Facts`, `Meta`. **Holds:** nothing.
- **Props:** **`field`** (field) — a field or link of the record · **`value`** (expr) · **`label`** (text) — default from the field; required with value or span · **`sub`** (expr|element) — a grey line: "in 3 days" · **`span`** (list:field) — [start, end] dates, times or timestamps shown as one value "07:00–19:30"
- **Example:** the [RecordPage](#recordpage) example.

### Meta
- ≤ 6 Facts on one line under the record title. **In:** `RecordPage`. **Holds:** `Fact`.
- **Props:** none.
- **Example:** the [RecordPage](#recordpage) example.

### Line
- One money line. **In:** `Breakdown`. **Holds:** nothing.
- **Props:** **`field`** (field) · **`value`** (expr) · **`label`** (text) · **`kind`** (enum: subtotal / total / due / credit / minus / part) — subtotal = a ruled line ("Net"); total and due 16/700; credit = a credit note; minus = discounts, cuts, paid, shown with −; part = an "of which" line, indented, not part of the sum · **`each`** (expr) — a keyed preview output (one line per key, labelled with the key and its base: "VAT 13 % on €5,060.00") or a to-many link of the record (one line per row: `field` = the row's money field, `label` = the row's label field); zero lines hidden · **`when`** (expr) — as on `Section`
```jsx
<RecordPage type="quote">
  <Section title="Lines">
    <Breakdown of={record}>
      <Line each={record.lines} field="net_cents" label="description" />
    </Breakdown>
  </Section>
  <Form run="quote.revise_lines">
    <Preview id="pv" />
    <Lines field="lines"><Field name="qty" /><Field name="unit_cents" /></Lines>
    <Breakdown of={pv}>
      <Line each={pv.net_by_rate} />
      <Line field="net" kind="subtotal" label="Net" />
      <Line each={pv.vat_by_rate} />
      <Line field="total" kind="total" label="Total" />
    </Breakdown>
    <Submit />
  </Form>
</RecordPage>
```

### Column
- One column. **In:** `Table`. **Holds:** nothing.
- **Props:** **`field`** (field) — own field, 1-hop path, rollup, or a to-many path (a capped list of ≤ 3 linked titles + "+N") · **`value`** (fn) — a row function: r => … · **`label`** (text) — default from the field; required with value · **`importance`** (enum: auto / primary / eyebrow / secondary / detail, default auto) — auto: first column primary, money and status secondary, the rest detail · **`total`** (enum: sum / count) — money/number fields; over the whole filtered set; nulls count as 0, "—" when every value is null · **`sort`** (boolean) — default true for fields · **`sortBy`** (field) — with value: the field a value column sorts by · **`sub`** (fn) — a row function for a sub-line: r => …; a number or amount read from a field is written with that field's label ("Staff needed 9"), never bare — for assigned of needed write value={r => <Ratio value={r.assigned} need={r.needed} />}
- **Example:** recipe 3 (`Trip.list`) in [recipes.md](recipes.md).

### Item
- The row/card template: eyebrow, title, ≤ 2 sub facts, value, ≤ 2 Badges after the title (a List row and a Board card; a Timeline bar and a DayGrid block draw its eyebrow, title, lead, sub, value and detail, no Badge); the status pill is automatic. **In:** `List`, `Board`, `Timeline`, `DayGrid`. **Holds:** `Badge`.
- **Props:** **`title`**\* (expr) — the human name (15 px); the row's link · **`eyebrow`** (expr) — default the type's code field (display.code), unless the title starts with it or another slot shows it; in containers ≥ 1200 px a muted inline prefix on the title line ("EV-1187 · Team lunch"), above the title only in card form · **`lead`** (expr) — a time or a person; the time leads when the Query is sorted by it; on a Timeline bar or DayGrid block a time lead is the bar's own time (never repeated), a person the first fact · **`sub`** (list:expr) — ≤ 2 render expressions or to-many paths; bare numbers show their label; null facts are hidden · **`value`** (expr) — number or money, right · **`detail`** (list:expr) — ≤ 3 short labelled facts on one muted line ("3 sessions · approved €410"); dropped first on phone; null facts are hidden
- **Example:** recipe 1 (`Departure.today`) in [recipes.md](recipes.md).

### Group
- Groups rows by a key; an Action child is a group verb whose subject is the group's key record. **In:** `Table`, `List`. **Holds:** `Action`.
- **Props:** **`by`**\* (field) — a path, optional @day/@week/@month/@bucket (Overdue / Today / This week / Later) · **`order`** (list:string) — key order; default: an enum key's declared order, else by first appearance in the Query's sort · **`collapsed`** (list:string) — keys collapsed by default · **`measure`** (field) — numeric field summed in the header
```jsx
<Page title="Payments by kind" lead={pay}>
  <Query id="pay" type="charge" sort={["kind", "due_on"]} />
  <Section>
    <Table of={pay}>
      <Column field="trip_ref" importance="primary" />
      <Column field="open_cents" total="sum" />
      <Group by="kind" measure="open_cents" />
    </Table>
  </Section>
</Page>
```

### Expand
- The row opens in place; may hold one scoped Query whose where reads the row (read on open). **In:** `Table`. **Holds:** a child function returning `Section`.
- **Props:** none.
- **Example:** recipe 3 (`Trip.list`) in [recipes.md](recipes.md).

### Check
- A leading 24 px check control (44 px hit area on phone). **In:** `Table`, `List`. **Holds:** nothing.
- **Props:** **`run`**\* (name:action) — the transition to done · **`undo`**\* (name:action) — its inverse · **`done`** (name:condition) — the condition that counts as checked (default: run's target state)
```jsx
<Page title="Yard checklist" lead={yard}>
  <Query id="yard" type="departure" sort="report_at" />
  <Section>
    <List of={yard}>
      {d => <Item lead={d.report_at} title={d.coach} sub={[d.drivers]} />}
      <Check run="departure.check_out" undo="departure.undo_check_out" />
    </List>
  </Section>
</Page>
```

### BulkAction
- Runs a one-subject action over the selected rows (invoke_many, ≤ 100). **In:** `Table`. **Holds:** nothing.
- **Props:** **`run`**\* (name:action)
- **Example:** recipe 3 (`Trip.list`) in [recipes.md](recipes.md).

### Empty
- The empty state: a title, a reason and ≤ 1 Action; several Empty parts, first match wins. **In:** `Page`, `RecordPage`, `Section`, `Tab`, `Table`, `List`, `Board`, `Matrix`, `Timeline`, `DayGrid`, `Form`. **Holds:** text, `Value`, `Link`, `Action`.
- **Props:** **`title`**\* (text) · **`when`** (expr) — as on `Section`
```jsx
<Page title="Charges for a trip" lead={ch}>
  <State id="tr" type="trip" />
  <Query id="ch" type="charge" sort="due_on" where={c => c.trip == tr} />
  <Toolbar><Control state={tr} label="Trip" /></Toolbar>
  <Section>
    <Table of={ch}>
      <Column field="kind" importance="primary" />
      <Empty title="Choose a trip" when={tr == null}>Pick a trip above to see what its hirer owes.</Empty>
      <Empty title="No charges">This trip has no charges yet.</Empty>
    </Table>
  </Section>
</Page>
```

### Rows
- One row set of a Matrix: records, enum values or one fixed row. **In:** `Matrix`. **Holds:** a child function returning `Cell`.
- **Props:** **`of`** (ref:Query) — one row per record, every record of the Query (read whole, never a page) · **`values`** (field) — an enum field "type.field": one row per value · **`label`** (text) — one fixed row (row = null) · **`sub`** (fn) — row → a sub-line · **`cells`** (ref:Query) — this row set's cell records (overrides the Matrix's) · **`cellKey`** (list:field)
- **Example:** the [Matrix](#matrix) example.

### Cell
- One matrix cell: ≤ 4 inline facts (text, never interactive) and one target. **In:** `Rows`. **Holds:** `Ratio`, `Value`, `Text`, `Action`.
- **Props:** **`opens`** (expr) — a record: the cell opens it (with `list` or Actions: the popover's first row is "Open <record>") · **`list`** (expr) — a list Aggregate cell: the records the popover lists (not counted as a fact) · **`current`** (boolean)
- **Example:** the [Matrix](#matrix) example.

### Span
- A shaded stretch of time: dates cover whole days, timestamps cover the stretch. **In:** `Timeline`, `DayGrid`. **Holds:** nothing.
- **Props:** **`of`**\* (ref:Query) · **`kind`**\* (enum: blocked / available) · **`lane`** (field) — no lane = all lanes · **`start`**\* (field) · **`end`**\* (field) · **`label`** (field) — the visible text, e.g. "Time off: course"
- **Example:** the [DayGrid](#daygrid) example.

### Field
- Places one model input; everything else about it is the model's. **In:** `Section`, `Split`, `Tab`, `Form`, `Lines`, `Step` (needs a `Form` ancestor). **Holds:** nothing.
- **Props:** **`name`**\* (name:input)
- **Example:** the [Form](#form) example.

### Lines
- An array input as a lines grid: a row per line, a column per sub-input; `<Value of={line.<property>} />` shows each line's own value, a property of the row it inserts (the action's `presentation.preview.rows`). **In:** `Section`, `Split`, `Tab`, `Form`, `Step` (needs a `Form` ancestor). **Holds:** `Field`, `Value`.
- **Props:** **`field`**\* (name:input)
```jsx
<RecordPage type="trip">
  <Form run="trip.add_extras">
    <Preview id="pv" />
    <Lines field="extras"><Field name="description" /><Field name="qty" /><Field name="unit_cents" /><Value of={line.net_cents} /></Lines>
    <Breakdown of={pv}><Line field="extras_cents" label="Extras" /><Line field="total_cents" kind="total" /></Breakdown>
    <Submit />
  </Form>
</RecordPage>
```

### Step
- Layout of one model step (pure layout): its Fields must belong to that step. **In:** `Section`, `Split`, `Tab`, `Form` (needs a `Form` ancestor). **Holds:** `Field`, `Lines`, `Section`, `Candidates`.
- **Props:** **`name`**\* (name:step)
- **Example:** the [Form](#form) example.

### Submit
- A submit button for one of the model's submits. **In:** `Section`, `Split`, `Tab`, `Form` (needs a `Form` ancestor). **Holds:** nothing.
- **Props:** **`mode`** (name:submit) — the model submit this button runs. Required when the action declares more than one submit: write one `<Submit mode=…>` per submit the screen offers (they render in written order); a `<Submit>` without `mode` is legal only when the action has a single submit (or none), and then runs it. The forward submit (one whose `next` runs a forward transition, e.g. Send) is primary and is the sticky phone bar's button; Save draft is secondary
- **Example:** the [Form](#form) example.

### Candidates
- Choose a related record with each option's rule outcome (Form-as-screen; generated dialogs render it by themselves). **In:** `Section`, `Split`, `Tab`, `Form`, `Step` (needs a `Form` ancestor). **Holds:** nothing.
- **Props:** **`field`**\* (name:input) · **`facts`** (list:field) — candidate fields shown; may read the subject (model)
```jsx
<Page title="Assign a driver" params={{departure: "departure"}}>
  <Record id="dep" type="departure" key={param("departure")} />
  <Form run="departure.assign_driver">
    <Preview id="pv" />
    <Section title={dep} of={dep}>
      <Candidates field="driver_id" facts={["licence", "week_driving_min"]} />
      <Submit />
    </Section>
  </Form>
</Page>
```

## Helpers

| Helper | Returns | Meaning |
|---|---|---|
| `today()` | date | the org clock's date (demo clock included) |
| `now()` | timestamp | the org clock's instant |
| `addDays(date, n)` | date |  |
| `weekOf(date)` | week | `.start`, `.end`: the Monday–Sunday week holding the date; a State's `weekdays` narrow the days it shows |
| `monthOf(date)` | month | a month key for @month Aggregates |
| `days(weekOrRange)` | list:date | the shown days of a week/range State (its weekdays) |
| `weeks(rangeOrMonth)` | list:week | the weeks a range or month State touches, from the org's week start: Matrix columns or an Aggregate's `over` at week grain |
| `months(range)` | list:month | the months a range State touches: Matrix columns or an Aggregate's `over` at month grain |
| `hours(day)` | list:timestamp | the hours of a date State's day in the org zone (23 or 25 on a DST day): Matrix columns or an Aggregate's `over` at hour grain |
| `param(name)` | string\|record | a declared view parameter |
| `me()` | record | the reader (`.name`, `.id`); in predicates only as `r.link == me()` |
| `greeting()` | text | "Good morning, Lena" in the org zone |
| `plural(n, one, many)` | text | the word only, by the count: `plural(n, "stop", "stops")` is "stop" for 1 and "stops" otherwise. Write the number beside it: `<Text><Value of={n} /> <Value of={plural(n, "stop", "stops")} /></Text>` reads "1 stop" / "12 stops" |
| `percent(part, whole)` | number | a computed share written as a percentage: `percent(rewash.value, washed.value)` with 29 of 1,000 is "2.9%" ("2,9 %" in Danish). One argument is a ratio, a fraction of one: `percent(0.029)` is "2.9%"; never multiply by 100 yourself. To 1 place unless the percentage is whole ("50%"); a Stat's `scale` sets the places. A whole of 0 or none is no value ("—"). A field that holds a ratio declares `scale: {as: "percent"}` in the model instead |

## Handles

| Handle | Returns | Meaning |
|---|---|---|
| `q.total` | number | rows the collection shows (all filters) |
| `q.count(id)` | number | a Preset's count or a condition's row count |
| `q.amount(condition)` | number | Σ of the condition's amount ("7 open positions") |
| `q.sum(field[, condition])` | number\|money | a declared total |
| `q.preset` | string | the active preset id |
| `q.current` | record | the current (→ Next) record, or null |
| `a.at(k1[, k2])` | number\|list | an Aggregate cell |
| `a.value` | number | an Aggregate without by |
| `a.next(key)` | key | the first key ≥ key with a non-zero value |
| `a.peak(k)` | number\|money | the largest cell along the time key (an over series or a bucketed key), the other key fixed: the busiest day of a range |
| `a.peakAt(k)` | date | the period a.peak(k) falls on (the earliest of equal peaks); null when nothing is counted |
| `a.free(k, capacity)` | number\|money | capacity − a.peak(k), never below 0; past the capacity it reads 0 and is short by the excess on the peak day ("0 short 1 on Sat 28 Nov", critical, its row railed) |
| `a.free(k, capacity, extra)` | number\|money | a capacity that differs by day: on each day capacity + extra.at(k, day) − a.at(k, day), the least of them, never below 0; short by the deficit on the tightest day. extra is a count or sum Aggregate keyed like a (owned stock + what is hired in that day) |
| `s.start / s.end` | date | week and range States |
| `pv.<output>` | typed | a Preview output |
| `line.<output>` | typed | inside Lines: the line's own value (a `preview.rows` property of the row it inserts) |
| `input.<name>` | typed | inside a Form: the current input value |

**Scope names:** record (RecordPage), input (Form), line (Lines), child-function parameters, every data element id, every State id. An id may not shadow a helper or a scope name.
