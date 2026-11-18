<!-- GENERATED from the Spring UI schema and the example pages by `bun run generate:skills` — do not edit -->

# Spring UI components

Spring UI 3.4 has **56 names an author writes, and no others**: 2 page templates, 5 data elements, 30 components, 19 typed parts. An element the catalog does not name, or a prop an element does not take, is refused at compile time. A screen whose layout is the data (a calendar, a schedule, a board of nested marks) is not drawn with these names: it is a Region, [regions.md](regions.md). Each entry says what the name is, where it may sit (**In**), what it may hold (**Holds**), its props, and gives an example page. The language behind them is [language.md](language.md); verbs and forms are [verbs-and-forms.md](verbs-and-forms.md).

**Reading an entry.** **In: any layout** = `Page`, `RecordPage`, `Section`, `Split`, `Columns`, `Tab`, `Form`. **Holds: any component** = every component except `Action` (written where its parent allows it). A prop marked \* is required. Prop types: `text` a literal string · `expr` a render expression in `{…}` · `predicate` a row arrow `{r => …}` (thin narrowing) · `fn` a row function `{r => …}` · `field` a field or link name as a string · `id` the element's name in expressions · `ref:Query` a data element's id as `{q}` · `name:action` (`name:type`, `name:view`, …) a model name as a string · `enum: a / b` one of the listed values · `list:x` an array `{["a", "b"]}` · `object` `{{key: value}}` · `element` JSX such as `<Text>…</Text>`.

**Not drawn yet.** The kernel draws every name and prop below except those marked **Not drawn yet**. A name or prop marked "refused at compile" is refused when the page compiles: write the form its entry names instead. One marked "it compiles and the page shows nothing for it" is accepted but not drawn: leave it out of a page that depends on it. The marks are read from the kernel's own schema, so they go away when the kernel draws the construct.

**The example model.** Every example compiles against one model, a city library, and omits the `{/* @catalog spring-ui */}` first line each file needs. `patron`: card_no, name, email, phone, a link `home_branch`, joined_on, expires_on (dates), loan_limit, loans_out, id_scan (a file), and a status machine applied → active → suspended (or expired, closed) run by patron_activate, patron_suspend, patron_reinstate, patron_close; patron_renew_card moves expires_on. `loan`: links `patron` and `copy`, item, format (enum), out_on, due_on, returned_on, renewals, days_late (derived), a machine `state` out → returned (or lost) run by loan_return and loan_mark_lost, loan_renew, and the condition `is_overdue` (level critical, on due_on, amount days_late, fix loan_renew). `hold`: a waiting → ready → collected machine `state` (hold_make_ready, hold_collect, hold_undo_collect), item, shelf, patron_name, card_no, ready_at (timestamp), ready_on (its date, derived), pickup_by and the condition `is_expiring` (level critical, on pickup_by, fix hold_extend). `fine`: links `patron` and `loan`, ref, reason (enum), due_on, amount_cents, paid_cents, open_cents (derived), status (enum) and the condition `is_overdue` (level critical, on due_on, amount open_cents, fix fine_record_payment). `branch`, `book` (name, author, format, cover: a file), `copy` (book, branch, barcode, shelf). Actions with page-facing concerns: patron_join (a create), loan_return (preview outputs days_late and late_fee), fine_record_payment (a guard on the amount), hold_assign_copy (a ref input `copy_id` with candidates). For the components that need more: `book_suggestion` (item, copies, a status machine suggested → approved → sent → received → catalogued, or rejected), `expense_report` (submitted_by, version_no, items_count, total_cents, to-many `items` of `expense_item`: line_key, description, qty, net_cents, vat_pct) with report_approve, report_revise_items (a lines input `items`) and report_add_items (a lines input `receipts`); `room` (name, seats), `room_booking` (room, purpose, date, starts_at, ends_at), `room_closure` (reason, date, starts_at, ends_at), `room_day` (a room × date row the bookings keep: booked, seats, wanted, extra_seats and the condition is_overbooked); and, beside the library, a sports club's `team` (name, squad) and `team_day` (team, date, available, needed and the condition is_short). Your page names your model's types, fields, conditions and actions, exactly.

## Index

| Kind | Names |
|---|---|
| Page templates | [Page](#page), [RecordPage](#recordpage) |
| Data elements | [Query](#query), [Record](#record), [Aggregate](#aggregate), [State](#state), [Preview](#preview) |
| Components | [Section](#section), [Split](#split), [Columns](#columns), [Grid](#grid), [Tabs](#tabs), [Toolbar](#toolbar), [Value](#value), [Text](#text), [Badge](#badge), [Callout](#callout), [Stat](#stat), [Ratio](#ratio), [Meter](#meter), [Chart](#chart), [Facts](#facts), [Breakdown](#breakdown), [History](#history), [Lifecycle](#lifecycle), [Table](#table), [List](#list), [Filters](#filters), [Search](#search), [Control](#control), [PeriodNav](#periodnav), [Files](#files), [Compare](#compare), [Action](#action), [Link](#link), [Form](#form), [Region](#region) |
| Typed parts | [Preset](#preset), [Flag](#flag), [Tab](#tab), [Series](#series), [Fact](#fact), [Meta](#meta), [Line](#line), [Column](#column), [Item](#item), [Group](#group), [Expand](#expand), [Check](#check), [BulkAction](#bulkaction), [Empty](#empty), [Field](#field), [Lines](#lines), [Step](#step), [Submit](#submit), [Candidates](#candidates) |

## Page templates

### Page
- A screen. Data elements first, then content; Action children are header verbs. **In:** top level (one per file). **Holds:** `Query`, `Record`, `Aggregate`, `State`, any component, `Action`, `Empty`.
- **Props:** **`title`**\* (text|element) — the page title; text, an expression or a record (its title) · **`subtitle`** (text|element) — the author's line under the title (e.g. the order of the queue) · **`eyebrow`** (expr) — a quiet label above the title: the record's code or id, never the display title (eyebrow={record.code}); a blank or withheld value leaves it out · **`lead`** (ref:Query|list:ref:Query) — the collection(s) that drive the header summary and the nav badge; required when the page has more than one Query; a list sums the attention of each ("2 over · 3 unassigned"). The count is in the type's noun ("23 loans") · **`params`** (object) — the view's parameters and their types: {patron: "patron"}; param() reads them · **`audience`** (list:group) — groups that may open this view (checked against the org's groups); others do not see it in nav or links (reads stay policy-checked) · **`navLabel`** (text) — the page's name in the app's nav, breadcrumb and Home (literal text); default: the title when it is text, else the view name's last part (`Sales.dashboard` → "Dashboard"). A reader who sees two nav entries with the same name sees each followed by its audience ("Dashboard · Sales"), else its page name · **`icon`** (text) — the page's glyph in the app's nav (literal text): a kebab-case Lucide name from the platform's pinned set (@forspring/ui's icons), e.g. "users". Default, by what the page is: a RecordPage none; else none. None: the nav draws no glyph. Give each nav page its own
```jsx
<Page title="Fines" subtitle="Oldest open fine first" lead={fs}>
  <Query id="fs" type="fine" sort="due_on"><Flag condition="is_overdue" /></Query>
  <Section><Table of={fs}><Column field="ref" importance="primary" /><Column field="open_cents" total="sum" /></Table></Section>
</Page>
```

### RecordPage
- A record's screen. Reads param("id"); the record is in scope as `record`; Action children are record header verbs. **In:** top level (one per file). **Holds:** `Query`, `Record`, `Aggregate`, `State`, `Meta`, any component, `Action`, `Empty`, `Flag`.
- **Props:** **`type`**\* (name:type) — the record's type · **`title`** (list:expr) — the heading as parts of the record, drawn in order with a dot between them (title={[record.first_name, record.last_name]}: "Anna Berg"); the title is a name, never an id (no "LN-2041 · Night Trains": the id is a muted `Fact` in `Meta`); a blank part drops with its separator, a part the reader may not read leaves quietly, and with none left the record's own title stands. The browser tab reads the same parts. Without it: the record's title · **`subtitle`** (text|element) · **`eyebrow`** (expr) — as on `Page` · **`lead`** (ref:Query) · **`audience`** (list:group) · **`icon`** (text) — as on `Page`
```jsx
<RecordPage type="patron">
  <Meta><Fact field="card_no" /><Fact field="expires_on" /><Fact field="loans_out" /></Meta>
  <Action run="patron_activate" role="primary" />
  <Section title="Details"><Facts><Fact field="home_branch" /><Fact label="Member" span={["joined_on", "expires_on"]} /></Facts></Section>
</RecordPage>
```

## Data elements

### Query
- The collection: a declared slice of one type plus the reader's working state (preset, facets, search, sort, page, selection). Served by one server batch. **In:** `Page`, `RecordPage`, `Section`; in `Section` only inside `Expand`. **Holds:** `Preset`, `Flag`.
- **Props:** **`id`**\* (id) — the name in expressions; may not shadow a helper or scope name · **`type`**\* (name:type) · **`where`** (predicate) — thin narrowing (see language.md, predicates) · **`sort`** (field|list:field) — "due_on", "-version" or a list; own or 1-hop fields; a link sorts by its target's title; an enum by its declared values; a named condition sorts true first with "-" ("-is_full"); blanks always last · **`page`** (number, default 25) — rows per page (≤ 200); 0 = unpaged (≤ 500). · **`search`** (list:field|boolean) — own or 1-hop text fields; default the type's searchable; false turns Search off · **`facets`** (list:field) — ≤ 3 enum, set<enum> or link fields shown as facet chips · **`current`** (predicate) — the first row in sort order that matches and carries no critical condition is the current ("→ Next") row · **`mine`** (predicate) — a row predicate under the rules of `where` (`b => b.member.user == me()`; to reach a person through a link, declare a lookup on the type with visibility over_readable and compare it with `me()`): a row it holds carries an own-row marker (`mine`) a Region can draw as "yours". Never a reader id; absent when the predicate is false, blank or decided by a value the reader cannot read; computed in the reader's own read of the Query · **`nameBy`** (field) — the field that names a row in the queue header ("pick-up time passed: 07:00 H-03"), e.g. the shelf; default the title, or the eyebrow when the title is longer than 24 characters
```jsx
<Page title="Fines" lead={fs}>
  <Query id="fs" type="fine" sort="due_on" page={14} facets={["reason"]}>
    <Preset id="open" label="Open" where={f => f.status in ["open", "part_paid"]} default />
    <Preset id="all" label="All" />
    <Flag condition="is_overdue" />
  </Query>
  <Toolbar><Filters of={fs} /><Search of={fs} /></Toolbar>
  <Section><Table of={fs}><Column field="ref" importance="primary" /><Column field="due_on" /><Column field="open_cents" total="sum" /></Table></Section>
</Page>
```

### Record
- One record by key. **In:** `Page`, `RecordPage`. **Holds:** nothing.
- **Props:** **`id`**\* (id) · **`type`**\* (name:type) · **`key`**\* (expr) — the record's key: param("patron"), another Record's link, or a date for a per-day type whose ids are dates; a type with prefixed ids ("day_…") is read by its date field with a Query and a State auto="single" (a date key is refused)
```jsx
<Page title={pt} params={{patron: "patron"}}>
  <Record id="pt" type="patron" key={param("patron")} />
  <Query id="pLoans" type="loan" sort="-out_on" where={l => l.patron == pt} />
  <Section title="Loans"><Table of={pLoans}><Column field="item" importance="primary" /><Column field="out_on" /><Column field="state" /></Table></Section>
</Page>
```

### Aggregate
- One grouped read, keyed; read with `.at()`. **In:** `Page`, `RecordPage`. **Holds:** nothing.
- **Props:** **`id`**\* (id) · **`type`** (name:type) — the type to aggregate; or use `of` · **`of`** (ref:Query) — inherit the Query's type and the rows it reads: its base where (incl. bound States) and the window the components drawing it show (a State's weekdays), never its page, preset or search; may only narrow · **`op`**\* (enum: count / sum / min / max / avg / list) · **`property`** (field) — the measured field (sum/min/max/avg); for list: the link whose target records are listed — without it the aggregated rows themselves are listed (they are the popover records) · **`by`** (field|list:field) — ≤ 2 keys: own or 1-hop fields; date keys may end in @day, @week, @month, @bucket; with span the day key is written `date` · **`on`** (field) — the date or timestamp that places a row on a day (default: the single date key) · **`span`** (list:field) — [start, end]: the row counts on every day it covers; not drawn yet, refused at compile (tick dzn) · **`over`** (expr) — a day series: days(week) · **`window`** (object) — {last: n} buckets for @week/@month keys · **`where`** (predicate) · **`facts`** (list:field) — with op=list: ≤ 3 fields of each listed record (the rows, or the `property` targets), shown in the popover · **`decimals`** (number) — not set on an Aggregate: refused at compile (page.structure.unsupported). Write decimals where the value is drawn: on Value, Stat, Fact, Column, Item or Ratio. **Not drawn yet:** refused at compile.
```jsx
<Page title="Hold shelf" lead={holds}>
  <Query id="holds" type="hold" sort="pickup_by" page={50} />
  <Aggregate id="byState" type="hold" op="count" by="state" />
  <Aggregate id="copies" type="book_suggestion" op="sum" property="copies" where={r => r.status == "approved"} />
  <Grid>
    <Stat label="Handed over" value={byState.at("collected")} />
    <Stat label="Copies approved to buy" value={copies.value} />
  </Grid>
</Page>
```

### State
- A value the reader chooses; kept in the URL under its id. **In:** `Page`, `RecordPage`. **Holds:** nothing.
- **Props:** **`id`**\* (id) · **`type`**\* (enum|name:type: date / week / range / month / boolean / number / string / enum) — or an ObjectType name = a ref State · **`initial`** (expr) — required for date/week/range/month; a range's is {from: <date>, days: <whole number>}, e.g. {{from: today(), days: 14}} (never {start, end}); wins over auto when not null · **`options`** (list:object) — enum States: [{value, label}] · **`where`** (predicate) — ref States without `from`: the choices · **`sort`** (field|list:field) — ref States without `from`; a link sorts by its target's title, as a Query's · **`from`** (ref:Query) — ref States: the choices are this Query's rows — its where, active preset, chosen facets and search, in its sort, never just its page — so the picker lists what the page lists; no where or sort of its own · **`auto`** (enum: single / first / second / current) — ref States: pick the only choice, the first/second in sort, or the `from` Query's current row; not drawn yet, refused at compile (tick qx0): nothing is picked until the reader chooses (a filter State reads null as "all") · **`weekdays`** (list:string) — week/range States: the days shown, e.g. ["mon",…,"sat"]; default every day. days(week) and PeriodNav (its label and day strip) show only these
```jsx
<Page title="On loan this week" lead={wk}>
  <State id="week" type="week" initial={weekOf(today())} weekdays={["mon", "tue", "wed", "thu", "fri", "sat"]} />
  <Query id="wk" type="loan" sort="due_on" page={0} where={l => l.out_on <= week.end && l.due_on >= week.start} />
  <Toolbar><PeriodNav state={week} span="week" /></Toolbar>
  <Section><Table of={wk}><Column field="item" importance="primary" /><Column field="due_on" /></Table></Section>
</Page>
```

### Preview
- The Form action's live preview outputs as `pv.<output>` (typed, policy-checked). **In:** `Form` (needs a `Form` ancestor). **Holds:** nothing.
- **Props:** **`id`**\* (id)
```jsx
<RecordPage type="loan">
  <Form run="loan_return">
    <Preview id="pv" />
    <Grid><Stat label="Days late" value={pv.days_late} /><Stat label="Late fee" value={pv.late_fee} /></Grid>
    <Submit />
  </Form>
</RecordPage>
```

## Components

### Section
- A titled region; a top-level Section is one card with its title in the header. **In:** `Page`, `RecordPage`, `Section`, `Split`, `Columns`, `Tab`, `Expand`, `Form`, `Step`. **Holds:** any component, `Action`, `Empty`, `Query`, `Flag`, `Field`, `Lines`, `Step`, `Submit`, `Candidates`.
- **Props:** **`title`** (text|element|expr) — text, or a record expression (its title) · **`of`** (expr) — the section's record: scope for verbs (incl. its Toolbar), default record of Facts; when null, a record title hides, components that read it render nothing and the Section's Empty speaks · **`when`** (expr) — render expression; false hides the element (UX only, never security). A withheld operand makes it false. · **`audience`** (list:group) — groups it is drawn for (checked against the org's groups); a reader in none of them is not served it at all — no title, no lock rows (owners and admins see every one); decided on the server, not a data boundary (reads stay policy-checked)
```jsx
<RecordPage type="patron">
  <Query id="pf" type="fine" sort="due_on" where={f => f.patron == record} />
  <Section title="Fines" of={record}>
    <Action run="patron_suspend" />
    <Table of={pf}><Column field="reason" importance="primary" /><Column field="amount_cents" total="sum" /></Table>
  </Section>
</RecordPage>
```

### Split
- Main + aside, exactly two children; the aside is sticky at ≥ 1024 px when it holds a Form summary. In a Form, the side holding its inputs (Field, Lines, Step, Submit, Candidates) is sized by its fields and the result side takes the rest, in either order. **In:** any layout. **Holds:** `Section`, `Split`, `Columns`, `Grid`, `Tabs`, `Toolbar`, `Value`, `Text`, `Badge`, `Callout`, `Stat`, `Ratio`, `Meter`, `Chart`, `Facts`, `Breakdown`, `History`, `Lifecycle`, `Table`, `List`, `Filters`, `Search`, `Control`, `PeriodNav`, `Files`, `Compare`, `Link`, `Form`, `Field`, `Lines`, `Step`, `Submit`, `Candidates`.
- **Props:** none.
- **Example:** recipe 1 (`patron_record`) in [recipes.md](recipes.md).

### Columns
- 2–4 different children side by side when the container is ≥ 960 px; stacked otherwise. **In:** any layout. **Holds:** `Section`, `Split`, `Columns`, `Grid`, `Tabs`, `Toolbar`, `Value`, `Text`, `Badge`, `Callout`, `Stat`, `Ratio`, `Meter`, `Chart`, `Facts`, `Breakdown`, `History`, `Lifecycle`, `Table`, `List`, `Filters`, `Search`, `Control`, `PeriodNav`, `Files`, `Compare`, `Link`, `Form`.
- **Props:** none.
- **Example:** recipe 6 (`loan_week`) in [recipes.md](recipes.md).

### Grid
- Like items only: Stat or Files tiles; the tile count picks the tracks at every width — wide, 2–5 tiles one track each (3 tiles → 3, never 3 of 4), 6 → 3, 7 or more → 4; on phone 2 (3 for 3, 1 for 5 or 7) — so the first row has no empty track and no later row is an orphan; tiles never stretch; a withheld tile reflows the rest left. **In:** any layout. **Holds:** `Stat`, `Files`.
- **Props:** none.
- **Example:** the [Aggregate](#aggregate) example.

### Tabs
- A tool's views (a builder, an editor), or record sections you group yourself; a long RecordPage draws its sections as tabs on its own. **In:** any layout. **Holds:** `Tab`.
- **Props:** none.
```jsx
<RecordPage type="patron">
  <Query id="pf" type="fine" sort="due_on" where={f => f.patron == record} />
  <Tabs>
    <Tab label="Details"><Facts><Fact field="email" /><Fact field="phone" /></Facts></Tab>
    <Tab label="Fines" lead={pf}><Table of={pf}><Column field="reason" importance="primary" /><Column field="amount_cents" /></Table></Tab>
  </Tabs>
</RecordPage>
```

### Toolbar
- Inputs and verbs for the page or a section (a Section's first child). **In:** any layout. **Holds:** `Search`, `Filters`, `Control`, `PeriodNav`, `Action`.
- **Props:** none.
- **Example:** the [Query](#query) example.

### Value
- One field or typed expression, shown the one way its type and container show it. **In:** `Page`, `RecordPage`, `Section`, `Split`, `Columns`, `Tab`, `Text`, `Badge`, `Callout`, `Empty`, `Form`, `Lines`. **Holds:** nothing.
- **Props:** **`of`**\* (expr) — a field, link, aggregate cell, list, handle or typed expression · **`decimals`** (number) — places the figure is written to (0–6): display only, the stored value and any expression over it keep full precision; zeros pad above its own places; a money keeps its currency places unless set; a text, date or flag is refused · **`change`** (boolean, default false) — this is a difference: show the sign (+15) and the `better` tone
```jsx
<Page title="Fines" lead={fs}>
  <Query id="fs" type="fine" sort="due_on"><Flag condition="is_overdue" /></Query>
  <Section title="Overdue">
    <Text><Value of={fs.count(f => f.is_overdue)} /> fines are overdue, <Value of={fs.where(f => f.is_overdue).sum(f => f.open_cents)} /> still open.</Text>
  </Section>
</Page>
```

### Text
- Prose with inline values. **In:** any layout. **Holds:** text, `Value`, `Link`, `Badge`.
- **Props:** **`when`** (expr) — as on `Section`
```jsx
<Page title="Hold shelf" subtitle={<Text>Ready for pick-up · now <Value of={now()} /></Text>} lead={holds}>
  <Query id="holds" type="hold" sort="pickup_by" />
  <Section><List of={holds}>{h => <Item lead={h.pickup_by} title={h.item} />}</List></Section>
</Page>
```

### Badge
- A neutral mark with text ("Rush", "Repeat"); attention marks come from Flags, never from a Badge. **In:** `Page`, `RecordPage`, `Section`, `Split`, `Columns`, `Tab`, `Text`, `Callout`, `Item`, `Form`. **Holds:** text, `Value`.
- **Props:** **`when`** (expr) — as on `Section`
```jsx
<Page title="Patrons" lead={patrons}>
  <Query id="patrons" type="patron" sort="name" />
  <Section><List of={patrons}>{p => <Item title={p.name} sub={[p.card_no]}><Badge when={p.loans_out >= p.loan_limit}>At loan limit</Badge></Item>}</List></Section>
</Page>
```

### Callout
- A note the author writes: info, or a warning about the screen's content (not a refusal, not a data exception). **In:** any layout. **Holds:** text, `Value`, `Link`, `Badge`.
- **Props:** **`tone`** (enum: info / warn / critical, default info) · **`title`** (text) · **`condition`** (name:condition) — in a RecordPage: shown only while this named condition of the page record holds; its tone is the condition's level (critical, warn; good and neutral → info); its text is what you write inside, else the condition's label and amount; nothing (never a hint) when the reader may not read what the condition reads; refused without a page record · **`when`** (expr) — as on `Section`
```jsx
<Page title="Suggestions to approve" lead={q}>
  <Query id="q" type="book_suggestion" sort="suggested_on" where={r => r.status == "suggested"} />
  <Callout tone="warn" title="Suggestions lapse after 30 days">Approve or reject each one before 30 days have passed since the date in the last column.</Callout>
  <Section><Table of={q}><Column field="item" importance="primary" /><Column field="suggested_on" /></Table></Section>
</Page>
```

### Stat
- A KPI tile: label, value, optional delta, foot line and sparkline. **In:** `Page`, `RecordPage`, `Section`, `Split`, `Columns`, `Grid`, `Tab`, `Form`. **Holds:** `Ratio`.
- **Props:** **`label`**\* (text) · **`value`** (expr) — number, money or duration; or a Ratio child · **`compare`** (enum: previous_period / same_period_last_year) — shifts the time key of the Aggregate in value; a period in progress is set against the same elapsed span of the earlier one ("on 1–10 Nov"); for the whole earlier period, show its total in a Ratio child instead · **`better`** (enum: up / down) — default from the field's `better` · **`context`** (element) — the foot line · **`trend`** (ref:Aggregate) — keyed by time → sparkline · **`decimals`** (number) — decimal places a number value is written to (0–6), the word a number property's `scale: { decimals }` uses; default: its field's or Aggregate's decimals; a computed figure (`a.value * 100 / b.value`) that is not whole to 1 place, and arithmetic keeps the unit of its typed side · **`when`** (expr) — as on `Section`
- **Example:** recipe 6 (`loan_week`) in [recipes.md](recipes.md).

### Ratio
- "assigned / needed": the numerator in 600 ink, "/ needed" muted. Its tone comes from its two numbers, never from a prop: `of` is a whole (capacity), `need` a requirement. **In:** `Page`, `RecordPage`, `Section`, `Split`, `Columns`, `Tab`, `Stat`, `Form`. **Holds:** nothing.
- **Props:** **`value`**\* (expr) · **`of`** (expr) — a capacity the value fills (available, a limit): over it is critical "over by n"; at or under it the pair is neutral; a target to reach is `need`, not `of`; a literal (`of={2400}`) is in the value's unit (minutes: "40 h") · **`need`** (expr) — what is needed (players per match, seats per room): below it is short — critical "short n"; at it met — a check; above it over — warn "over by n"; a literal is in the value's unit · **`decimals`** (number) — as on `Value` · **`remainder`** (text) — with `need`: the words for what is still missing below it, after its count — `remainder="to check"` reads "26 / 35 · 9 to check" where the default reads "short 9" (lines not yet checked are not short); literal text
- **Example:** recipe 2 (`room_days`) in [recipes.md](recipes.md).

### Meter
- A bar with its number; tone from the field's `thresholds` (model), healthy = neutral ink. **In:** any layout. **Holds:** nothing.
- **Props:** **`value`**\* (expr) · **`of`**\* (expr)
- **Example:** recipe 1 (`patron_record`) in [recipes.md](recipes.md).

### Chart
- A chart by question; the platform picks bars/lines/stacking. **In:** any layout. **Holds:** `Series`.
- **Props:** **`shows`**\* (enum: trend / composition / comparison / capacity) · **`limit`** (expr) — capacity line · **`when`** (expr) — as on `Section`
- **Example:** recipe 6 (`loan_week`) in [recipes.md](recipes.md).

### Facts
- Key–value facts of a record. **In:** any layout. **Holds:** `Fact`.
- **Props:** **`of`** (expr) — the record (default: the nearest Section's `of`, else `record`); a link (record.space) carries the target's title only, so its Facts are refused: read it with a Record and name that
- **Example:** the [RecordPage](#recordpage) example.

### Breakdown
- A money summary; lines in written order (it shows, it does not add). **In:** any layout. **Holds:** `Line`.
- **Props:** **`of`**\* (expr) — a record or a Preview
- **Example:** recipe 1 (`patron_record`) in [recipes.md](recipes.md).

### History
- A record's changes: its transitions and its history: true fields (PA-8), or a declared event log. **In:** any layout. **Holds:** nothing.
- **Props:** **`of`**\* (expr) · **`fields`** (list:field) — the type's opt-in history fields · **`log`** (field) — a to-many link of an event type to read instead of the transition log · **`at`** (field, default at) — the event's date or timestamp · **`by`** (field, default by) — who: a field or a link (its record's title) · **`action`** (field, default action) · **`note`** (field, default note)
```jsx
<RecordPage type="patron">
  <Lifecycle of={record} />
  <Split>
    <Section title="What happened"><History of={record} /></Section>
    <Section title="Documents"><Files of={record} /></Section>
  </Split>
</RecordPage>
```

### Lifecycle
- Where a record is in its process, as a stepper: the machine's main path in order, each step done (✓ and its date), current or next; a taken branch (cancelled, rejected, lost, or a second good end) replaces the rest of the path, drawn only when the record took it; in a narrow place one line, "Step 3 of 5 · Packed", and the next step. **In:** any layout. **Holds:** nothing.
- **Props:** **`of`**\* (expr)
- **Example:** the [History](#history) example.

### Table
- Aligned columns over a collection. **In:** any layout. **Holds:** `Column`, `Action`, `BulkAction`, `Group`, `Expand`, `Check`, `Empty`.
- **Props:** **`of`**\* (ref:Query|ref:Aggregate|expr) — a Query or an Aggregate (one row per key); a Preview array (pv.items) is not drawn yet, refused at compile · **`opens`** (expr|string, default auto) — "auto" (the type's record view; rows of a type without one, Aggregate rows and preview rows do not open), a view name, a record expression (opens its view) or "none" · **`picks`** (ref:State) — a ref State of the row type: a row click sets it (master–detail); not drawn yet, refused at compile: choose the record with a Control on the ref State, or let the rows open their record (opens) · **`verbs`** (enum: flag / job / rest / menu, default flag) — flag (default): a verb is visible only on rows whose flag it fixes (the condition's `fix`), as a bordered button; job: the first offered verb is the page's per-row job — a quiet text button (the action's `shortLabel`: "Record") on every eligible row, bordered on flagged rows; `rest` is a synonym of job; menu: every verb in ⋯. A queue (a Query with `current`, or a Check) defaults to job. A verb whose preCheck guard fails never shows at rest. ⋯ appears only when it holds a verb (opening the record is the title's job, not a menu item); a job column may show different verbs per row (the first offered differs by state); a create with no subject may be a row verb, its inputs filled from the row. · **`when`** (expr) — as on `Section`
- **Example:** recipe 5 (`loan_list`) in [recipes.md](recipes.md).

### List
- Free rows with the card anatomy (queues, run sheets). **In:** any layout. **Holds:** a child function returning `Item`, `Action`, `Group`, `Check`, `Empty`.
- **Props:** **`of`**\* (ref:Query|expr) — a Query or an Aggregate (one row per key); a Preview array is not drawn yet, refused at compile · **`opens`** (expr|string, default auto) — as on `Table` · **`picks`** (ref:State) — not drawn yet, refused at compile: choose the record with a Control on the ref State, or let the rows open their record (opens) · **`verbs`** (enum: flag / job / rest / menu, default flag) — as on `Table` · **`fade`** (fn) — a row predicate (`h => h.state == "waiting"`): a row it holds draws faded (still readable) and is left out of every group aggregate: its count, its sum and its capacity load; a header with faded rows adds "· n not counted". A predicate, not a hue, because "does not count" is about the totals · **`when`** (expr) — as on `Section`
- **Example:** recipe 3 (`holds_today`) in [recipes.md](recipes.md).

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
- **Props:** **`state`**\* (ref:State) · **`span`**\* (enum: day / week / fortnight / month) — the step: a date State steps by day, a week by week, a month by month, a range by its length (fortnight) or by a week or a month (its length kept; a range of whole months keeps its months) · **`counts`** (ref:Aggregate|list:ref:Aggregate) — keyed by date: a day strip with badges in the header's unit; a list is summed (short and overbooked days) · **`skip`** (enum: none / empty, default none) — empty: arrows step to the next day with a count
- **Example:** the [State](#state) example.

### Files
- A record's attachments: tiles, lightbox, Add when the reader may attach. **In:** `Page`, `RecordPage`, `Section`, `Split`, `Columns`, `Grid`, `Tab`, `Form`. **Holds:** nothing.
- **Props:** **`of`**\* (expr) · **`kinds`** (list:string) — attachment kinds of the type (incl. kinds backed by a child type, PA-14)
- **Example:** the [History](#history) example.

### Compare
- Two versions of a record with aligned lines, signed deltas and totals. **Not drawn yet:** refused at compile. **In:** any layout. **Holds:** nothing.
- **Props:** **`a`**\* (expr) · **`b`**\* (expr) · **`lines`**\* (field) — a to-many link · **`key`**\* (field) — unique per record: a field or a link (the line's item); a duplicate is a render refusal naming it · **`value`**\* (field) — the line money field: per-line signed delta, totals per side, difference · **`fields`** (list:field) — line fields compared and shown; a line is Changed when any differs · **`facts`** (list:field) — header fields aligned above the lines, each with its change · **`total`** (field) — the record's own money or number total (e.g. incl. VAT): a/b and the difference
<!-- refused: page.structure.unsupported(Compare) -->
```jsx
<Page title="Compare report versions" params={{report: "expense_report"}}>
  <Record id="rp" type="expense_report" key={param("report")} />
  <State id="va" type="expense_report" where={o => o.number == rp.number} sort="-version_no" auto="second" />
  <State id="vb" type="expense_report" where={o => o.number == rp.number} sort="-version_no" auto="first" />
  <Toolbar><Control state={va} label="Version" /><Control state={vb} label="Against" /></Toolbar>
  <Section title="Items">
    <Compare a={va} b={vb} lines="items" key="line_key" value="net_cents" total="total_cents" fields={["qty", "net_cents"]} facts={["items_count"]} />
  </Section>
</Page>
```

### Action
- The one verb element: runs an action (generated dialog) or opens a view. **In:** `Page`, `RecordPage`, `Section`, `Tab`, `Toolbar`, `Table`, `List`, `Group`, `Empty`, `Region`. **Holds:** nothing.
- **Props:** **`run`** (name:action) — an action · **`view`** (name:view) — a view to open · **`params`** (object|fn) — with view: named parameters, or a row function (r => ({patron: r.patron})) so a Form page can be a row's fix verb · **`label`** (text) — required with view; refused with run (the model's label) · **`role`** (enum: primary / secondary) — primary: may sit on verbs that exclude each other by legality, the first offered one is primary. secondary: a header verb that stays a button beside the primary instead of going to More (at most 2, in written order; a third stays in More, and so does a destructive one; one the record's state rules out now still goes to More with its reason). Only header verbs read it: on a row, card or menu verb it has no effect · **`with`** (object|fn) — input presets: {{patron_id: pt}} or a row function r => ({format: r.format}); null clears a fill · **`rest`** (boolean) — false: never the at-rest row verb · **`when`** (expr|fn) — render expression; false hides the element (UX only, never security). A withheld operand makes it false. On a Table/List row verb, a row function l => l.state == "out" is its row condition: the verb is offered only on the rows it holds for.
```jsx
<RecordPage type="patron">
  <Action run="patron_close" />
  <Action run="patron_suspend" />
  <Action run="patron_activate" role="primary" />
  <Action view="patron_list" label="All patrons" />
</RecordPage>
```

### Link
- Inline navigation to a view or a record. **In:** `Page`, `RecordPage`, `Section`, `Split`, `Columns`, `Tab`, `Text`, `Callout`, `Empty`, `Form`. **Holds:** text.
- **Props:** **`view`** (name:view) · **`params`** (object) · **`record`** (expr) — opens the record's view
```jsx
<RecordPage type="loan">
  <Section title="Patron"><Text>Borrowed by <Link record={record.patron}>the patron</Link> · <Link view="patron_list">all patrons</Link></Text></Section>
</RecordPage>
```

### Form
- An action's form as the screen; a scope: any layout inside; Field/Lines/Step/Submit/Candidates/Preview need a Form ancestor. **In:** any layout. **Holds:** `Preview`, `Section`, `Split`, `Columns`, `Grid`, `Tabs`, `Toolbar`, `Value`, `Text`, `Badge`, `Callout`, `Stat`, `Ratio`, `Meter`, `Chart`, `Facts`, `Breakdown`, `History`, `Lifecycle`, `Table`, `List`, `Filters`, `Search`, `Control`, `PeriodNav`, `Files`, `Compare`, `Link`, `Form`, `Field`, `Lines`, `Step`, `Submit`, `Candidates`, `Empty`.
- **Props:** **`run`**\* (name:action) · **`with`** (object) — input presets, e.g. {{book_suggestion_id: param("suggestion")}}; a null value leaves the input to its model default · **`view`** (name:view) — the view a successful submit opens, with `params` (built like a Link: a view the reader may not open, or a param with no value, opens nothing). Without it the Form resets in place, or opens the record the action navigates to (`presentation.refresh: "navigate"`) · **`params`** (object) — with view: its named parameters, from the page's scope ({{patron: pt}})
```jsx
<Page title="New patron">
  <Form run="patron_join">
    <Step name="card"><Field name="name" /><Field name="card_no" /><Field name="home_branch_id" /></Step>
    <Step name="contact"><Field name="email" /><Field name="phone" /></Step>
    <Submit />
  </Form>
</Page>
```

### Region
- One region of a page that the app’s own code draws, over the data the page declares and reads for the reader. **In:** `Page`, `RecordPage`, `Section`. **Holds:** `Action`, `Group`.
- **Props:** **`code`**\* (id) — the bundle module that draws the region: a plain name ([a-z][a-z0-9_-]*), the file views/<view>/<code>.region.js of the bundle, applied and versioned with it. The module is self-contained (one default export, no import) and is checked when the bundle is applied: tokens only (no raw colour), no network, storage, eval or frames; a module that fails is not applied, and an unapplied Region draws as a fault (page.region.unapplied) · **`data`**\* (object) — what the region receives, by name: data={{name: q, …}}, each entry a Query or an Aggregate of the page (a name is [a-z][a-zA-Z0-9_]*). A Query is read per reader exactly as a Table reads it (its policies, withheld values and verb legality): its rows, its total, whether a whole read was cut, its verbs; an Aggregate with no group is its one figure, a grouped one its groups as rows. A read that fails, is withheld or waits for a choice is that value, never an empty list · **`fields`** (object) — the personal or secret properties a rows entry reads, by data name: fields={{bookings: "phone, email"}}; every other property of the row type is read without naming, a sensitive one only when named here (it is masked per reader and its read is logged) · **`decimals`** (object) — the places a rows entry writes its figures with, by data name: decimals={{bookings: 0}}; every money and number field of each row of that entry is written to that many places (0 to 6), as `decimals` on a Fact or Column does: display only, the stored value and every expression keep full precision. An entry that is no rows entry, or a value outside 0 to 6, is refused (page.structure.prop_type) · **`calendar`** (expr) — a CX calendar function of the catalog, by name (calendar={opening_hours}; never a value): with `range`, the page gets each day of the range with its weekly hours, closed on each of its holidays the reader may read (one read under the reader’s rights); a day the week leaves out reads closed. A calendar with params is refused (nothing gives them a value) · **`range`** (ref:State|expr) — with calendar: the days it gives hours for, a week, range or month State or a period expression; over 366 days page.days is the fault page.region.days_limit
- **Example:** the page in [regions.md](regions.md#the-element).

## Typed parts

### Preset
- A saved view of the collection; Filters shows it as a chip with its live count. **In:** `Query`. **Holds:** nothing.
- **Props:** **`id`**\* (id) · **`label`**\* (text) · **`where`** (predicate) — no where = all rows · **`default`** (boolean, default false) — at most one; none = the first Preset · **`short`** (text) — the chip label at phone width ("Due 14 d"); a chip never clips its label · **`verbs`** (enum: flag / job / menu) — the collection's verbs mode while this preset is active ("Ready to issue" → job, "Overdue" → flag)
- **Example:** the [Query](#query) example.

### Flag
- Marks rows that need attention, from a named condition of the row type. In a RecordPage (or a Section of one) it flags the page record: a line in the header, with the condition's level and words from the model, while the condition holds; nothing when it does not hold or when the reader may not read what it reads. **In:** `RecordPage`, `Query`, `Section`; in `Section` only inside `RecordPage`. **Holds:** nothing.
- **Props:** **`condition`**\* (name:condition) — a named condition (Part A PA-3); meaning, level, on, amount and label come from the model. In a Query it marks the rows; in a RecordPage (or its Section) it flags the page record
- **Example:** the [Query](#query) example.

### Tab
- One tab. **In:** `Tabs`. **Holds:** `Section`, `Split`, `Columns`, `Grid`, `Tabs`, `Toolbar`, `Value`, `Text`, `Badge`, `Callout`, `Stat`, `Ratio`, `Meter`, `Chart`, `Facts`, `Breakdown`, `History`, `Lifecycle`, `Table`, `List`, `Filters`, `Search`, `Control`, `PeriodNav`, `Files`, `Compare`, `Link`, `Form`, `Action`, `Empty`, `Field`, `Lines`, `Step`, `Submit`, `Candidates`.
- **Props:** **`label`**\* (text) · **`lead`** (ref:Query) — the collection whose total is the tab's count badge (default: the tab's first collection) · **`audience`** (list:group) — as on `Section`
- **Example:** the [Tabs](#tabs) example.

### Series
- One series of a Chart. **In:** `Chart`. **Holds:** nothing.
- **Props:** **`of`**\* (ref:Aggregate) · **`label`**\* (text)
- **Example:** recipe 6 (`loan_week`) in [recipes.md](recipes.md).

### Fact
- One fact. **In:** `Facts`, `Meta`. **Holds:** nothing.
- **Props:** **`field`** (field) — a field or link of the record · **`value`** (expr|element) — an expression, or an element (`<Text>`, `<Value>`, `<Ratio>`) drawn as the value · **`label`** (text) — default from the field; required with value or span · **`sub`** (expr|element|list:expr) — a grey line: "in 3 days"; a list of parts (`sub={[c.kind, c.email, c.phone]}`) is one line of them, each in its own form, and a blank part drops with its separator ("Hotel · bookings@…", never "Hotel · —"); a `<Text>` sentence is for your own words around values, where a blank value keeps its dash · **`span`** (list:field) — [start, end] dates, times or timestamps shown as one value "07:00–19:30" · **`format`** (enum: tel / maps / mailto) — a text field written as a phone number (tel: a tap-to-call link), an address (maps: a tap-to-map link) or an email address (mailto: a mailto: link): the same words, the link built on the server; a blank, withheld or failed value has none; a number, date or flag is refused · **`decimals`** (number) — as on `Value`
- **Example:** the [RecordPage](#recordpage) example.

### Meta
- ≤ 6 Facts on one line under the record title. **In:** `RecordPage`. **Holds:** `Fact`.
- **Props:** none.
- **Example:** the [RecordPage](#recordpage) example.

### Line
- One money line. **In:** `Breakdown`. **Holds:** nothing.
- **Props:** **`field`** (field) · **`value`** (expr) · **`label`** (text) · **`kind`** (enum: subtotal / total / due / credit / minus / part) — subtotal = a ruled line ("Net"); total and due 16/700; credit = a credit note; minus = discounts, cuts, paid, shown with −; part = an "of which" line, indented, not part of the sum · **`each`** (expr) — a to-many link of the record (one line per row: `field` = the row's money field, `label` = the row's label field); zero lines hidden. A breakdown by key (net by VAT rate) is a page Aggregate drawn as a Table: an action output is one value, never keyed · **`when`** (expr) — as on `Section`
```jsx
<RecordPage type="expense_report">
  <Aggregate id="byRate" type="expense_item" op="sum" property="net_cents" by="vat_pct" where={l => l.report == record} />
  <Section title="Items">
    <Breakdown of={record}>
      <Line each={record.items} field="net_cents" label="description" />
    </Breakdown>
  </Section>
  <Section title="Net by VAT rate">
    <Table of={byRate}><Column field="vat_pct" /><Column label="Net" value={r => byRate.at(r)} /></Table>
  </Section>
  <Form run="report_revise_items">
    <Preview id="pv" />
    <Lines field="items"><Field name="qty" /><Field name="unit_cents" /></Lines>
    <Breakdown of={pv}><Line field="total" kind="total" label="Total" /></Breakdown>
    <Submit />
  </Form>
</RecordPage>
```

### Column
- One column. **In:** `Table`. **Holds:** nothing.
- **Props:** **`field`** (field) — own field, 1-hop path or rollup. A back-link (the rows of another type that point at this row) is not a field: count it with value={r => r.<link>.count()} (a child type the reader may not read gives a withheld cell, never 0), or list the rows in a Query of the child type · **`value`** (fn) — a row function: r => … · **`label`** (text) — default from the field; required with value · **`importance`** (enum: auto / primary / eyebrow / secondary / detail, default auto) — auto: first column primary, money and status secondary, the rest detail · **`total`** (enum: sum / count) — money/number fields; over the whole filtered set; nulls count as 0, "—" when every value is null · **`decimals`** (number) — as on `Value` · **`format`** (enum: tel / maps / mailto) — as on `Fact` · **`sort`** (boolean) — default true for fields · **`sortBy`** (field) — with value: the field a value column sorts by · **`sub`** (fn) — a row function for a sub-line: r => …; a number or amount read from a field is written with that field's label ("Renewals 2"), never bare — for available of needed write value={r => <Ratio value={r.available} need={r.needed} />}
- **Example:** recipe 5 (`loan_list`) in [recipes.md](recipes.md).

### Item
- The row/card template: eyebrow, title, ≤ 2 sub facts, value, ≤ 2 Badges after the title (a List row); the status pill is automatic. **In:** `List`. **Holds:** `Badge`.
- **Props:** **`title`**\* (expr) — the human name (15 px); the row's link · **`eyebrow`** (expr) — default the type's code field (display.code), unless the title starts with it or another slot shows it; in containers ≥ 1200 px a muted inline prefix on the title line ("EV-1187 · Team lunch"), above the title only in card form · **`lead`** (expr) — a time or a person; the time leads when the Query is sorted by it · **`sub`** (list:expr) — ≤ 2 render expressions or to-many paths; bare numbers show their label; null facts are hidden · **`value`** (expr) — number or money, right · **`decimals`** (number) — as on `Value` · **`detail`** (list:expr) — ≤ 3 short labelled facts on one muted line ("3 sessions · approved €410"); dropped first on phone; null facts are hidden
- **Example:** recipe 3 (`holds_today`) in [recipes.md](recipes.md).

### Group
- Groups rows by a key an Action child is a group verb whose subject is the group's key record. **In:** `Table`, `List`, `Region`. **Holds:** `Action`.
- **Props:** **`by`**\* (field) — a path, optional @day/@week/@month/@bucket (Overdue / Today / This week / Later) · **`data`** (id) — only in a Region: the name of the data entry it groups (`<Region data={{loans: ln}}><Group data="loans" by="copy.branch" /></Region>`): that entry gets `groups`, as a Table's Group gives. Required in a Region, refused anywhere else · **`order`** (list:string) — key order; default: an enum key's declared order, else by first appearance in the Query's sort · **`collapsed`** (list:string) — keys collapsed by default · **`measure`** (field) — numeric field summed in the header · **`capacity`** (fn) — a number field of the group's key record, as a function of it (`g => g.seats`; the key `by` is a link): the group header draws a load bar of the `measure` summed over its rows that count against it ("6 of 8 seats"; "over by 2", danger past it). Needs `measure`; a faded row (the collection's `fade`) is left out
```jsx
<Page title="Fines by reason" lead={fs}>
  <Query id="fs" type="fine" sort={["reason", "due_on"]} />
  <Query id="hs" type="hold" sort="pickup_by" />
  <Section>
    <Table of={fs}>
      <Column field="ref" importance="primary" />
      <Column field="open_cents" total="sum" />
      <Group by="reason" measure="open_cents" />
    </Table>
  </Section>
  <Section title="Holds by branch">
    <List of={hs}>
      {h => <Item title={h.item} />}
      <Group by="branch"><Action run="branch_set_hours" /></Group>
    </List>
  </Section>
</Page>
```

### Expand
- The row opens in place; may hold one scoped Query whose where reads the row (read on open). **In:** `Table`. **Holds:** a child function returning `Section`.
- **Props:** none.
- **Example:** recipe 5 (`loan_list`) in [recipes.md](recipes.md).

### Check
- A leading 24 px check control (44 px hit area on phone). **In:** `Table`, `List`. **Holds:** nothing.
- **Props:** **`run`**\* (name:action) — the transition to done · **`undo`**\* (name:action) — its inverse · **`done`** (name:condition) — the condition that counts as checked (default: run's target state)
```jsx
<Page title="Hold shelf checklist" lead={shelf}>
  <Query id="shelf" type="hold" sort="pickup_by" />
  <Section>
    <List of={shelf}>
      {h => <Item lead={h.pickup_by} title={h.shelf} sub={[h.item]} />}
      <Check run="hold_collect" undo="hold_undo_collect" />
    </List>
  </Section>
</Page>
```

### BulkAction
- Runs a one-subject action over the selected rows (invoke_many, ≤ 100). **In:** `Table`. **Holds:** nothing.
- **Props:** **`run`**\* (name:action)
- **Example:** recipe 5 (`loan_list`) in [recipes.md](recipes.md).

### Empty
- The empty state: a title, a reason and ≤ 1 Action; several Empty parts, first match wins. **In:** `Page`, `RecordPage`, `Section`, `Tab`, `Table`, `List`, `Form`. **Holds:** text, `Value`, `Link`, `Action`.
- **Props:** **`title`**\* (text) · **`when`** (expr) — as on `Section`
```jsx
<Page title="Fines of a patron" lead={fs}>
  <State id="pt" type="patron" />
  <Query id="fs" type="fine" sort="due_on" where={f => f.patron == pt} />
  <Toolbar><Control state={pt} label="Patron" /></Toolbar>
  <Section>
    <Table of={fs}>
      <Column field="reason" importance="primary" />
      <Empty title="Choose a patron" when={isBlank(pt)}>Pick a patron above to see what they owe.</Empty>
      <Empty title="No fines">This patron has no fines.</Empty>
    </Table>
  </Section>
</Page>
```

### Field
- Places one model input; everything else about it is the model's. **In:** `Section`, `Split`, `Tab`, `Form`, `Lines`, `Step` (needs a `Form` ancestor). **Holds:** nothing.
- **Props:** **`name`**\* (name:input)
- **Example:** the [Form](#form) example.

### Lines
- An array input as a lines grid: a row per line, a column per sub-input; `<Value of={line.<property>} />` shows each line's own value, a property of the row it inserts (the action's `presentation.preview.rows`). **In:** `Section`, `Split`, `Tab`, `Form`, `Step` (needs a `Form` ancestor). **Holds:** `Field`, `Value`.
- **Props:** **`field`**\* (name:input)
```jsx
<RecordPage type="expense_report">
  <Form run="report_add_items">
    <Preview id="pv" />
    <Lines field="receipts"><Field name="description" /><Field name="qty" /><Field name="unit_cents" /><Value of={line.net_cents} /></Lines>
    <Breakdown of={pv}><Line field="receipts_cents" label="New receipts" /><Line field="total_cents" kind="total" /></Breakdown>
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
<Page title="Assign a copy" params={{hold: "hold"}}>
  <Record id="h" type="hold" key={param("hold")} />
  <Form run="hold_assign_copy">
    <Preview id="pv" />
    <Section title={h} of={h}>
      <Candidates field="copy_id" facts={["barcode", "shelf"]} />
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
| `weekday(date)` | number | the ISO weekday of a date, 1 Monday … 7 Sunday: `weekday(day) <= 5`, or a lookup keyed by weekday (`opening.weekday == weekday(day)`) |
| `yearsBetween(from, to)` | number | the whole calendar years from one date to another (an age, a tenure): `yearsBetween(r.born, today())` is 34 until the next birthday; a 29 Feb start counts the year on 28 Feb; negative when `to` is before `from` |
| `weekOf(date)` | week | `.start`, `.end`: the Monday–Sunday week holding the date; a State's `weekdays` narrow the days it shows |
| `monthOf(date)` | month | a month key for @month Aggregates |
| `days(weekOrRange)` | list:date | the shown days of a week/range State (its weekdays) |
| `weeks(rangeOrMonth)` | list:week | the weeks a range or month State touches, from the org's week start: an Aggregate's `over` at week grain |
| `months(range)` | list:month | the months a range State touches: an Aggregate's `over` at month grain |
| `hours(day)` | list:timestamp | the hours of a date State's day in the org zone (23 or 25 on a DST day): an Aggregate's `over` at hour grain |
| `param(name)` | string\|record | a declared view parameter |
| `me()` | record | the reader (`.name`, `.id`); in predicates only as `r.link == me()` |
| `greeting()` | text | "Good morning, Lena" in the org zone |
| `plural(n, one, many)` | text | the word only, by the count: `plural(n, "stop", "stops")` is "stop" for 1 and "stops" otherwise. Write the number beside it: `<Text><Value of={n} /> <Value of={plural(n, "stop", "stops")} /></Text>` reads "1 stop" / "12 stops" |
| `percent(part, whole)` | number | a computed share written as a percentage: `percent(renewed.value, loaned.value)` with 29 of 1,000 is "2.90 %". One argument is a ratio, a fraction of one: `percent(0.029)` is "2.90 %"; never multiply by 100 yourself. It is written to 2 places. A whole of 0 or none is no value ("—"). A field that holds a ratio declares `scale: {as: "percent"}` in the model instead |

## Handles

| Handle | Returns | Meaning |
|---|---|---|
| `q.count()` | number | rows the collection shows (all filters); never q.total |
| `q.count(r => r.condition)` | number | the rows that hold a named condition (or any row predicate) |
| `q.sum(r => r.field)` | number\|money | a total of a number or money field; over the rows that hold a condition: q.where(r => r.condition).sum(r => r.amount) |
| `q.preset` | string | the active preset id |
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
