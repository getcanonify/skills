# Recipes

Whole screens, each a real `.springml` file that compiles and renders: the pages of the coach-hire app the skill's examples use (its model is described at the top of [components.md](components.md)), byte for byte. Copy the shape, then rename to your model.

## 1. Today queue — a departure board

<!-- recipe: views/Departure.today.springml -->
```jsx
{/* @catalog spring-ui */}
<Page icon="bus" title="Today" subtitle="Departure board · in report-time order" lead={deps} params={{date: "date"}}>
  <State id="day" type="date" initial={param("date") ?? today()} />
  <Query id="deps" type="departure" sort="report_at" page={50} search={false} nameBy="coach" where={d => d.report_at == day} current={d => d.state == "due"}>
    <Flag condition="is_late" />
  </Query>
  <Aggregate id="byState" type="departure" op="count" by="state" where={d => d.report_at == day} />
  <Aggregate id="perDay" type="departure" op="count" by="report_at@day" />
  <PeriodNav state={day} span="day" counts={perDay} skip="empty" />
  <Grid>
    <Stat label="Out" value={byState.at("on_road")} context={<Text>on the road or on tour</Text>} />
    <Stat label="Back"><Ratio value={byState.at("back")} of={deps.total} /></Stat>
    <Stat label="To go" value={byState.at("due")} context={<Text><Value of={deps.count("is_late")} /> late</Text>} />
    <Stat label="Coaches today" value={deps.total} />
  </Grid>
  <Section title="Departures">
    <List of={deps}>
      {d => <Item lead={d.report_at} title={d.title} eyebrow={d.trip_ref} sub={[d.coach, d.drivers]} detail={[d.first_pickup, d.group_leader, d.leader_phone]} />}
      <Action run="departure.check_out" role="primary" when={day == today()} />
      <Action run="departure.check_in" when={day == today()} />
      <Action run="trip.depart" when={d => day == today() && d.state == "on_road"} />
      <Empty title="A quiet day">No coach leaves the depot this day.</Empty>
    </List>
  </Section>
</Page>
```

A **queue**: the List's Query is sorted by time and has `current`, so the first due departure that is not late gets "→ Next", the late ones carry a rail and a filled "Late · 20 min" pill (the `is_late` condition: meaning late, on `report_at`), and the header names them in a fixed form ("late: 07:15 SR-14 · next: 07:45 SR-05" — `nameBy="coach"` picks the coach as the name). A queue's verbs are `job` by default: "Check out" (the action's shortLabel) is quiet text on every row it is legal for, bordered on late rows (it is `is_late`'s fix); "Check in" is offered only on coaches on the road (the machine). The Stats read one grouped Aggregate (`byState.at("back")`) and the Query's own handles (`deps.total`, `deps.count("is_late")`), all in one read stage. `search={false}`: a 14-row board needs no search box. The board is one day: the page takes a `date` parameter (`params={{date: "date"}}`, given as `?date=2026-11-26`; a value that is not a date is refused), the State `day` starts at it or today, the Query and the Aggregate keep the departures that report on that day (`d.report_at == day` compares a timestamp with a day in the org's zone), and the PeriodNav in the header steps a day back, to today and on by writing `date` — the counts follow. The verbs are offered only on today (`when={day == today()}`).

## 2. Payments — a worklist with presets and totals

<!-- recipe: views/Charge.payments.springml -->
```jsx
{/* @catalog spring-ui */}
<Page icon="banknote" title="Payments" subtitle="Charges, payments and refunds · payments go to the oldest open charge first" lead={pay}>
  <Query id="pay" type="charge" sort="due_on" page={14}>
    <Preset id="due14" label="Due in 14 days" short="Due 14 d" where={c => c.due_on >= today() && c.due_on <= addDays(today(), 14) && ["open", "part_paid"].includes(c.status)} default />
    <Preset id="overdue" label="Overdue" where={c => c.is_overdue} />
    <Preset id="part_paid" label="Part paid" where={c => c.status == "part_paid"} />
    <Preset id="paid" label="Paid" where={c => c.status == "paid"} />
    <Preset id="all" label="All" />
    <Flag condition="is_overdue" />
  </Query>
  <Action run="charge.record_payment" role="primary" />
  <Grid>
    <Stat label="Overdue" value={pay.amount("is_overdue")} context={<Text><Value of={pay.count("is_overdue")} /> charges</Text>} />
    <Stat label="Open in this view" value={pay.sum("open_cents")} />
  </Grid>
  <Toolbar>
    <Filters of={pay} />
    <Search of={pay} />
  </Toolbar>
  <Section>
    <Table of={pay} verbs="job">
      <Column field="trip_ref" importance="primary" sub={c => c.trip_date} />
      <Column field="payer" />
      <Column field="kind" />
      <Column field="due_on" />
      <Column field="amount_cents" total="sum" />
      <Column field="paid_cents" total="sum" />
      <Column field="open_cents" total="sum" importance="secondary" />
      <Column field="status" />
      <Action run="charge.record_payment" />
      <Empty title="No charges yet">Charges appear here as soon as a trip is confirmed.</Empty>
    </Table>
  </Section>
</Page>
```

A **Table** because the reader works the columns: sorts, footer totals over every page ("Total of 14"), presets with live counts. The `due14` preset is the default; the `overdue` one narrows by the named condition (never by re-deriving "due before today and not paid" in the view). `verbs="job"` makes "Record" (the shortLabel of charge.record_payment) quiet text on every open row, bordered on overdue ones; the header Action is the same action with a charge picker (no charge in the page's scope). Overdue rows carry their mark, and the header counts them ("9 overdue") as a filter link.

## 3. A list with bulk verbs and an expandable row

<!-- recipe: views/Trip.list.springml -->
```jsx
{/* @catalog spring-ui */}
<Page icon="route" title="Trips" lead={trips}>
  <Query id="trips" type="trip" sort="-depart_date" page={14}>
    <Preset id="all" label="All" default />
    <Preset id="week" label="This week" where={t => t.depart_date <= weekOf(today()).end && t.return_date >= weekOf(today()).start} />
    <Preset id="awaiting" label="Awaiting reply" short="Awaiting" where={t => t.status == "quoted"} />
    <Preset id="wheelchair" label="Wheelchair" where={t => t.wheelchair_users > 0} />
  </Query>
  <Action view="Trip.new" label="New enquiry" role="primary" />
  <Toolbar>
    <Filters of={trips} />
    <Search of={trips} />
  </Toolbar>
  <Section>
    <Table of={trips}>
      <Column field="ref" importance="eyebrow" />
      <Column field="title" importance="primary" label="Trip" />
      <Column field="depart_date" />
      <Column field="destination" />
      <Column field="hirer" />
      <Column field="passengers" />
      <Column field="vehicle_class" />
      <Column field="total_cents" total="sum" importance="secondary" />
      <Column field="status" />
      <BulkAction run="trip.confirm" />
      <Action run="trip.confirm" />
      <Action run="trip.quote" />
      <Action run="trip.allocate" />
      <Expand>{t => <Section title="Charges">
        <Query id="tch" type="charge" sort="due_on" page={0} where={c => c.trip == t} />
        <Table of={tch}>
          <Column field="kind" importance="primary" />
          <Column field="due_on" />
          <Column field="amount_cents" total="sum" />
          <Column field="status" />
          <Empty title="No charges">This trip has no charges yet.</Empty>
        </Table>
      </Section>}</Expand>
      <Empty title="No trips yet">New enquiries appear here as soon as they are entered.</Empty>
    </Table>
  </Section>
</Page>
```

Presets for the common slices, Filters and Search in the Toolbar, a page header verb that opens another view (`view="Trip.new"` with its `label`). The row verbs are machine transitions: each row offers only the ones legal from its status, and with no flag naming them as a fix they sit in the row's ⋯. `BulkAction` adds a selection column on the rows that may be confirmed and runs `actions.invoke_many` over the selection. `Expand` opens a row in place with a Query scoped to that row (`c.trip == t`), read only when it opens.

## 4. Board — trips by status

<!-- recipe: views/Trip.board.springml -->
```jsx
{/* @catalog spring-ui */}
<Page title="Trips board" lead={live}>
  <Query id="live" type="trip" sort="depart_date" where={t => ["enquiry", "quoted", "confirmed", "allocated", "under_way", "completed"].includes(t.status)} />
  <Toolbar>
    <Search of={live} />
  </Toolbar>
  <Board of={live} by="status">
    {t => <Item title={t.title} eyebrow={t.ref} lead={t.depart_date} sub={[t.destination, t.vehicle_class]} value={t.passengers} detail={[t.hirer]} />}
  </Board>
</Page>
```

`by="status"` lays the lanes out in the machine's order (terminal lanes last); dragging a card, or its "Move to…" menu, runs the transition into that lane when it is legal for the reader and the card. The child function templates the card: eyebrow, title, lead, two sub facts, a value and detail.

This board keeps the six live states with its `where`. Drop the `where` and it takes all nine: the terminal and sink states (closed, lost, cancelled) fold into one last lane that counts each state and opens on demand. Each state in the fold is still a drop target, so dragging a quoted trip onto "Lost" runs `trip.mark_lost`. Do not fold states into a derived enum to fit a board: the derived lanes run no transition. A Board in a `Tab` with `<Filters of={q}>` over a Query with Presets draws the chips, and the lanes count the active preset.

## 5. Record page

<!-- recipe: views/Trip.record.springml -->
```jsx
{/* @catalog spring-ui */}
<RecordPage type="trip">
  <Query id="tch" type="charge" sort="due_on" page={0} where={c => c.trip == record} />
  <Aggregate id="charged" of={tch} op="sum" property="amount_cents" where={c => c.status != "cancelled"} />
  <Aggregate id="paid" of={tch} op="sum" property="paid_cents" where={c => c.status != "cancelled"} />
  <Aggregate id="open" of={tch} op="sum" property="open_cents" />
  <Meta>
    <Fact field="ref" />
    <Fact field="depart_date" />
    <Fact field="destination" />
    <Fact field="passengers" />
  </Meta>
  <Action run="trip.cancel" />
  <Action run="trip.quote" />
  <Action run="trip.allocate" />
  <Action run="trip.confirm" role="primary" />
  <Action run="trip.record_payment" />
  <Lifecycle of={record} />
  <Split>
    <Section title="Details">
      <Facts>
        <Fact field="hirer" />
        <Fact field="return_date" />
        <Fact field="vehicle_class" />
        <Fact field="wheelchair_users" />
        <Fact field="access_notes" />
        <Fact field="total_cents" />
      </Facts>
      <Link view="Trip.compare" params={{trip: record}}>Compare quote versions</Link>
      <Section title="Money">
        <Breakdown of={record}>
          <Line value={charged.value} label="Charged" />
          <Line value={paid.value} kind="minus" label="Paid" />
          <Line value={open.value} kind="due" label="Still to pay" />
        </Breakdown>
        <Meter value={paid.value} of={charged.value} />
      </Section>
      <Section title="Seats on one coach">
        <Meter value={record.passengers} of={57} />
      </Section>
    </Section>
    <Section title="Charges">
      <Table of={tch}>
        <Column field="kind" importance="primary" />
        <Column field="due_on" />
        <Column field="amount_cents" total="sum" />
        <Column field="open_cents" total="sum" />
        <Column field="status" />
        <Empty title="No charges">This trip has no charges yet.</Empty>
      </Table>
    </Section>
  </Split>
  <Section title="History">
    <History of={record} />
  </Section>
</RecordPage>
```

A `RecordPage` of a type is that type's record view: rows of trips anywhere open it. `Meta` puts ≤ 6 facts under the title; the header verbs are the machine's transitions (each shown only when legal, `role="primary"` on the forward one). Aggregates `of={tch}` inherit the scoped Query's where and read in its stage, beside it: they read its where, not its rows. `Breakdown` shows money lines in written order (it does not add); `Meter` shows paid against charged. `Lifecycle` draws the trip's path through its machine with the day it entered each state; `History` lists its transitions and its `history: true` fields' changes, newest first — both read the record's history under the reader's policy.

## 6. Week chart — a day series

<!-- recipe: views/Trip.week.springml -->
```jsx
{/* @catalog spring-ui */}
<Page icon="calendar-check" title="Fleet week" subtitle="Trips on the road each day · a trip counts on every day from departure to return" lead={wk}>
  <State id="week" type="week" initial={weekOf(today())} />
  <Query id="wk" type="trip" sort="depart_date" page={0} where={t => t.depart_date <= week.end && t.return_date >= week.start} />
  <Aggregate id="out" type="trip" op="count" by="date" span={["depart_date", "return_date"]} over={days(week)} />
  <Aggregate id="pax" type="trip" op="sum" property="passengers" by="date" span={["depart_date", "return_date"]} over={days(week)} />
  <Aggregate id="byClass" type="trip" op="count" by={["vehicle_class", "date"]} span={["depart_date", "return_date"]} over={days(week)} />
  <Aggregate id="leaving" type="trip" op="count" by="depart_date" over={days(week)} />
  <Aggregate id="value" type="trip" op="sum" property="total_cents" by="depart_date" over={days(week)} />
  <Aggregate id="going" type="trip" op="list" by="date" span={["depart_date", "return_date"]} over={days(week)} facts={["ref", "passengers", "total_cents"]} />
  <Grid>
    <Stat label="On the road today" value={out.at(today())} compare="previous_period" trend={out} />
    <Stat label="Passengers today" value={pax.at(today())} />
    <Stat label="Leaving today" value={leaving.at(today())} />
  </Grid>
  <Columns>
    <Section title="On the road each day">
      <Chart shows="composition"><Series of={byClass} label="Trips" /></Chart>
    </Section>
    <Section title="Trip value leaving each day">
      <Chart shows="trend"><Series of={value} label="Trip value" /></Chart>
    </Section>
  </Columns>
  <Section title="Trips this week">
    <Table of={wk}>
      <Column field="ref" importance="eyebrow" />
      <Column field="title" importance="primary" label="Trip" />
      <Column field="depart_date" />
      <Column field="return_date" />
      <Column field="destination" />
      <Column field="hirer" />
      <Column field="vehicle_class" />
      <Column field="passengers" />
      <Column field="status" />
      <Empty title="A quiet week">No trip is on the road this week.</Empty>
    </Table>
  </Section>
</Page>
```

A week State drives everything: the Query's narrowing, and every Aggregate's day series (`over={days(week)}`; `span` counts a trip on each day from departure to return). `out.at(today())` is today's cell; `compare` and `trend` give the Stat a delta and a sparkline. `Chart shows=` names the question — composition (stacked by the first key), trend — and the platform picks the marks.

## 7. Files — a record's attachment kinds

<!-- recipe: views/Trip.files.springml -->
```jsx
{/* @catalog spring-ui */}
<RecordPage type="trip">
  <Meta>
    <Fact field="ref" />
    <Fact field="depart_date" />
    <Fact field="destination" />
  </Meta>
  <Section title="Files">
    <Files of={record} />
  </Section>
</RecordPage>
```

`Files` shows the record's attachment kinds — each a child type declared on the record's type (`attachments.kinds`, see the canonify-objects-actions skill: a to-many link, the child's `file` property, an optional `kind` value, `facts`, `display: "hero"`). Each kind's files are tiles (an image's thumbnail, a PDF's mark) with a preview, under the reader's policy: a kind whose child type or file column the reader's role may not read is named "hidden for your role", never listed. "Add <kind>" is the child's create, filled with the record and kind, offered only when the reader may run it. After the kinds comes an **Attachments** block with the files attached to the record itself (an import's `files`, an upload to the record) when it has any — on a type with no kinds, always. Its "Add file" uploads to the record itself, held to the type's `attachments` `accept` / `max_bytes`, offered only when the reader may change the record (its `<type>.update` gate, the upload's own check) — no child type needed for one plain file list. `kinds={["damage_photo"]}` narrows it to those kinds, without Attachments. How files get there (seeding them with `objects.import`, uploading one): the canonify-objects-actions skill's [files](../../canonify-objects-actions/resources/files.md) page.

A `file` property needs no Files: name it in a `Fact`, `Meta`, `Column` or an `Item` slot and the reader sees the file by name — an image as a thumbnail, a PDF as a link that opens it, its size beside it — or "hidden for your role" when the policy withholds the property:

<!-- recipe: views/Trip.documents.springml -->
```jsx
{/* @catalog spring-ui */}
<RecordPage type="trip">
  <Query id="docs" type="trip_document" sort="-signed_on" page={0} search={false} where={d => d.trip == record} />
  <Query id="photos" type="trip_photo" page={0} search={false} where={p => p.trip == record} />
  <Meta>
    <Fact field="ref" />
    <Fact field="depart_date" />
  </Meta>
  <Section title="Documents">
    <Table of={docs}>
      <Column field="kind" importance="primary" />
      <Column field="signed_on" />
      <Column field="file" />
      <Empty title="No documents">No signed confirmation yet.</Empty>
    </Table>
  </Section>
  <Section title="Damage photos">
    <Table of={photos}>
      <Column field="note" importance="primary" />
      <Column field="file" />
      <Empty title="No damage photos">Nothing was found at check-in.</Empty>
    </Table>
  </Section>
</RecordPage>
```

## 8. Availability over a chosen range — free capacity per row

A derived property cannot read a page State, so a window written into derived SQL ("available next 7 days", "delivered in the last 6 weeks") is fixed for every reader. Put the window on the page instead: a range State the reader moves, Aggregates over it, and Columns that read those Aggregates per row.

<!-- recipe: views/Vehicle.availability.springml -->
```jsx
{/* @catalog spring-ui */}
<Page icon="bus-front" title="Coach availability" subtitle="Free coaches per class over the chosen days · a trip holds its coaches on every day it is away" lead={classes}>
  <State id="range" type="range" initial={{from: today(), days: 7}} />
  <Query id="classes" type="vehicle_class" sort="seats" page={0} search={false} />
  <Aggregate id="booked" type="trip" op="sum" property="vehicles_needed" by={["coach_class", "date"]} span={["depart_date", "return_date"]} over={days(range)} where={t => t.is_live} />
  <Aggregate id="leaving" type="trip" op="count" by="coach_class" where={t => t.is_live && t.depart_date >= range.start && t.depart_date <= range.end} />
  <Aggregate id="value" type="trip" op="sum" property="total_cents" by="coach_class" where={t => t.is_live && t.depart_date >= range.start && t.depart_date <= range.end} />
  <Toolbar><Control state={range} label="From" /><PeriodNav state={range} span="fortnight" /></Toolbar>
  <Section title="Coaches by class">
    <Table of={classes}>
      <Column field="name" importance="primary" />
      <Column field="own" label="Owned" />
      <Column label="Booked at peak" value={c => booked.peak(c)} sub={c => booked.peakAt(c)} />
      <Column label="Free every day" value={c => booked.free(c, c.own)} />
      <Column label="Leaving" value={c => leaving.at(c)} />
      <Column label="Booked value" value={c => value.at(c)} />
      <Empty title="No coaches">Add coach classes to plan the fleet.</Empty>
    </Table>
  </Section>
</Page>
```

- **"On no day may X exceed Y" needs the busiest day, not the range total.** `booked` is a day series keyed by the row and the day (`by={["coach_class", "date"]}` with `span` and `over={days(range)}`): a trip counts on every day it is away. Along that series `booked.peak(c)` is the class's busiest day, `booked.peakAt(c)` the day it falls on, and `booked.free(c, c.own)` what that day leaves free — never below 0: an over-booked class reads **0 · short 1 on Sat 28 Nov** in critical ink and its row wears the rail, so it stands out without a negative number.
- **A capacity that differs by day — `a.free(k, capacity, extra)`.** When what is available on a day is the owned stock plus what is hired in that day, a scalar capacity reports shortages that are not ones. Give the hire-ins their own day series keyed like the holds and pass it as the third argument: each day's capacity is `capacity` plus that day's `extra` cell, and the value is the least any day leaves free, short on the tightest day:

```jsx
{/* @catalog spring-ui */}
<Page icon="package" title="Kit" subtitle="Free per day against what we own and hire in" lead={kit}>
  <State id="range" type="range" initial={{from: today(), days: 7}} />
  <Query id="kit" type="kit_item" sort="name" page={0} search={false} />
  <Aggregate id="held" type="kit_reservation" op="sum" property="qty" by={["kit_item", "date"]} span={["out_on", "back_on"]} over={days(range)} />
  <Aggregate id="hired" type="kit_hire" op="sum" property="qty" by={["kit_item", "date"]} span={["from_on", "to_on"]} over={days(range)} />
  <Toolbar><Control state={range} label="From" /><PeriodNav state={range} span="week" /></Toolbar>
  <Table of={kit}>
    <Column field="name" importance="primary" />
    <Column field="owned_qty" label="Owned" />
    <Column label="Free every day" value={k => held.free(k, k.owned_qty, hired)} />
  </Table>
</Page>
```

  18 round tables hired in for Sat 31 Oct then cover the 18 held over the 140 owned: the column reads 0, not "0 short 18". `extra` must be a count or sum Aggregate keyed like `held` (the row, then the day) in the same unit (`aggregate_peak` otherwise). A handler's `peak_day` reads one type's busiest day, not capacity less holds per day: a booking guard that must agree with this column reads holds and hire-ins with `per_day` and checks each day of the request in a `for_each` over the holds' days (canonify-objects-actions, actions-handler-patterns: each day of a range).
- **A range sum is an Aggregate keyed by the row whose `where` reads the State** (`leaving`, `value`), read with `.at(c)`.
- **The reader moves the window**: `Control` types its first day, `PeriodNav` steps it by its length; every column re-reads. A measure the reader's policy withholds (`total_cents` for Dispatch) is withheld in every cell, never 0.
- **The guard says the same day.** An action that books capacity reads the busiest day with `read_aggregate` + `peak_day`, so its refusal names that day, not the first day of the request:

```json
{
  "op": "read_aggregate",
  "object": "trip",
  "where": { "coach_class_id": "$r.coach_class_id", "is_live": true, "id": { "neq": "$r.id" } },
  "sum": "vehicles_needed",
  "peak_day": { "span": ["depart_date", "return_date"], "from": "$r.depart_date", "to": "$r.return_date" },
  "as": "held"
}
```

then `"assert"` `$held.sum + $input.vehicles_needed` `lte` `$cls.own`, else `"Over-booked on {$held.day}: {$cls.name} owns {$cls.own}, {$held.sum} already booked, {$n} asked."` — "Over-booked on 24 Nov: Midi coach 35 owns 3, 3 already booked, 2 asked." (`examples/spring-b6` `trip.change_coaches`; the step is in canonify-objects-actions, Aggregates).

## 9. Par vs delivered over a chosen range

The same move on a record page: the record's lines, and per line what the deliveries in the chosen range needed and brought.

```jsx
{/* @catalog spring-ui */}
<RecordPage type="contract">
  <State id="range" type="range" initial={{from: addDays(today(), -42), days: 42}} />
  <Query id="lines" type="contract_line" sort="item" page={0} where={l => l.contract == record} />
  <Aggregate id="needed" type="delivery_line" op="sum" property="needed" by="item" where={d => d.contract == record && d.delivered_on >= range.start && d.delivered_on <= range.end} />
  <Aggregate id="delivered" type="delivery_line" op="sum" property="delivered" by="item" where={d => d.contract == record && d.delivered_on >= range.start && d.delivered_on <= range.end} />
  <Section title="Par vs delivered">
    <Toolbar><Control state={range} label="From" /><PeriodNav state={range} span="fortnight" /></Toolbar>
    <Table of={lines}>
      <Column field="item" importance="primary" />
      <Column field="par" />
      <Column label="Needed" value={l => needed.at(l.item)} />
      <Column label="Delivered" value={l => delivered.at(l.item)} />
      <Column label="Short" value={l => needed.at(l.item) - delivered.at(l.item)} />
    </Table>
  </Section>
</RecordPage>
```

The window is the page's (`{from: addDays(today(), -42), days: 42}` by default, six weeks back); the per-line facts that do not depend on it (`par`) stay fields of the line. A line's `needed` and `delivered` are fields of the delivery line (a derived `delivered_on` reads the delivery's date: the predicate narrows by an own field, never through a link).

## 10. Cover — a need against an actual, per row and day

<!-- recipe: views/Vehicle.cover.springml -->
```jsx
{/* @catalog spring-ui */}
<Page title="Coach cover" subtitle="Coaches allocated of needed per class and day · a trip needs its coaches on every day it is away" lead={trips}>
  <State id="week" type="week" initial={weekOf(today())} />
  <Query id="classes" type="vehicle_class" sort="seats" page={0} search={false} />
  <Query id="trips" type="trip" sort="depart_date" page={8} search={false} where={t => t.is_live && t.depart_date <= week.end && t.return_date >= week.start} />
  <Aggregate id="allocated" type="allocation" op="count" by={["trip.coach_class", "date"]} span={["from_date", "to_date"]} over={days(week)} where={a => a.is_live} />
  <Aggregate id="needed" type="trip" op="sum" property="vehicles_needed" by={["coach_class", "date"]} span={["depart_date", "return_date"]} over={days(week)} where={t => t.is_live} />
  <Aggregate id="going" type="trip" op="list" by={["coach_class", "date"]} span={["depart_date", "return_date"]} over={days(week)} where={t => t.is_live} />
  <Aggregate id="booked" type="allocation" op="count" where={a => a.is_live && a.from_date <= week.end && a.to_date >= week.start} />
  <Aggregate id="wanted" of={trips} op="sum" property="vehicles_needed" />
  <Toolbar><PeriodNav state={week} span="week" /></Toolbar>
  <Grid>
    <Stat label="Coaches allocated"><Ratio value={booked.value} need={wanted.value} /></Stat>
    <Stat label="Trips this week" value={trips.total} />
  </Grid>
  <Section title="Cover">
    <Matrix cols={days(week)} corner="Class">
      <Rows of={classes}>{(c, d) => <Cell list={going.at(c, d)}><Ratio value={allocated.at(c, d)} need={needed.at(c, d)} /></Cell>}</Rows>
      <Empty title="No coaches">Add coaches to plan the week.</Empty>
    </Matrix>
  </Section>
  <Section title="Trips this week">
    <Table of={trips}>
      <Column field="ref" importance="eyebrow" />
      <Column field="destination" importance="primary" />
      <Column field="depart_date" />
      <Column field="passengers" sub={t => t.wheelchair_users} />
      <Column field="vehicles_needed" />
    </Table>
  </Section>
</Page>
```

A **need against an actual** in every cell: `<Ratio value={…} need={…} />` reads "allocated of needed", and its tone comes from its two numbers — below its need it is **short** (critical "short 1", the cell tinted), at it **met** (a check), above it **over** (warn "over by 1"). Feed it two Aggregates keyed by the same row × column — here the allocations counted by the trip's class and day, and the coaches the live trips need summed by class and day; `span` counts both on every day they cover. Model a need as rows that carry it (a trip's `vehicles_needed`; for staff, one `staff_need` row per event × position with its `needed`), never as one column per position, so one Aggregate can sum it `by={["position", "date"]}`. `of=` is the other whole — a capacity: over it is critical, under it is neutral (`<Ratio value={booked} of={available} />`). The same Ratio draws the same tone in a `Stat`, a `Fact` value or a Column `value={r => <Ratio … />}`. A cell's `list` shows up to three of its records inline, each name whole (it wraps), then "+N more"; the popover lists them all. A Column whose `sub` is a number read from a field writes it with that field's label ("Wheelchair users 2"); for "assigned / needed" in one column write `value={r => <Ratio value={r.assigned} need={r.needed} />}`.

## 11. A long record page — section tabs and per-group sections

<!-- recipe: views/Trip.sheet.springml -->
```jsx
{/* @catalog spring-ui */}
<RecordPage type="trip" subtitle={<Text><Value of={record.hirer} /> · <Value of={record.destination} /></Text>}>
  <Record id="hr" type="hirer" key={record.hirer} />
  <Query id="coaches" type="allocation" sort="from_date" page={0} search={false} where={a => a.trip == record} />
  <Query id="duties" type="duty" sort={["date", "report_time"]} page={0} search={false} where={d => d.trip == record} />
  <Query id="charges" type="charge" sort="due_on" page={0} search={false} where={c => c.trip == record} />
  <Query id="quotes" type="quote" sort="-version" page={0} search={false} where={q => q.trip == record} />
  <Meta>
    <Fact field="ref" />
    <Fact field="depart_date" />
    <Fact field="passengers" />
    <Fact field="total_cents" />
  </Meta>
  <Action run="trip.allocate" />
  <Action run="trip.confirm" role="primary" />
  <Split>
    <Section title="Trip">
      <Facts>
        <Fact label="Out" span={["depart_date", "return_date"]} />
        <Fact field="vehicle_class" />
        <Fact field="wheelchair_users" />
        <Fact field="total_cents" />
      </Facts>
    </Section>
    <Section title="Hirer" of={hr}>
      <Facts>
        <Fact field="name" />
      </Facts>
    </Section>
  </Split>
  <Section title="Coaches" audience={["dispatch"]}>
    <Table of={coaches}>
      <Column field="vehicle" importance="primary" />
      <Column field="from_date" />
      <Column field="to_date" />
      <Column field="report_time" />
      <Empty title="No coach yet">Allocate a coach to this trip.</Empty>
    </Table>
  </Section>
  <Section title="Drivers">
    <Table of={duties}>
      <Column field="driver" importance="primary" />
      <Column field="role" />
      <Column field="date" />
      <Column field="report_time" />
      <Empty title="No drivers on duty">Nobody is rostered to this trip yet.</Empty>
    </Table>
  </Section>
  <Section title="Departure brief" audience={["driver"]}>
    <Facts>
      <Fact field="passengers" />
      <Fact field="wheelchair_users" />
      <Fact field="access_notes" />
    </Facts>
  </Section>
  <Section title="Money">
    <Tabs>
      <Tab label="Charges">
        <Table of={charges}>
          <Column field="kind" importance="primary" />
          <Column field="due_on" />
          <Column field="amount_cents" total="sum" />
          <Column field="open_cents" total="sum" />
          <Column field="status" />
          <Empty title="No charges">This trip has no charges yet.</Empty>
        </Table>
      </Tab>
      <Tab label="Quotes" audience={["dispatch"]}>
        <Table of={quotes}>
          <Column field="number" importance="primary" />
          <Column field="version" />
          <Column field="status" />
          <Column field="total_cents" />
          <Empty title="No quotes">This trip has not been quoted.</Empty>
        </Table>
      </Tab>
    </Tabs>
  </Section>
  <Section title="Files">
    <Files of={record} />
  </Section>
  <Section title="History" audience={["dispatch"]}>
    <History of={record} />
  </Section>
</RecordPage>
```

One record page for every role; each reader is served their part of it. **Overview, then tabs:** everything up to and including the first `Section`, `Split` or `Columns` (here the Split of Trip and Hirer) is the overview; when more than three titled Sections follow it, the page draws them as tabs on its own — one per Section, named by its title, counted by its first collection, the chosen one in the URL — so a record is not one long column. Write no `Tabs` for this; a top-level `Tabs` of your own replaces it. **Per-group sections:** `audience={["dispatch"]}` on a `Section` or `Tab` places it for those groups (owners and admins see every one); everyone else is not served it at all — no title, no "hidden" line. Use it for whole sections a role has no use for (Operations does not need the Proposal; Sales does not need the run sheet), and let the policy decide single values. **Withheld values are left out:** a `Fact` the reader may not read (here `total_cents` for a member) is not drawn; a `Facts`, `Section` or `Tab` with nothing left the reader may read (the Hirer for a driver, who may not read hirers; Money for a driver, who may not read charges and is not placed for Quotes) is not served, and a `Split` left with one side gives it the whole width. So a driver gets Trip, Drivers, Departure brief and Files stacked; Dispatch gets five tabs; the owner six. A collection withheld beside content the reader does read keeps its one "hidden for your role" note. `audience` is placement, not security: the reads stay policy-checked.
