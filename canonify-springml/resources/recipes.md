# Recipes

Whole screens, each a real `.springml` file that compiles and renders: the pages of the city library the skill's examples use (its model is described at the top of [components.md](components.md)), byte for byte. Copy the shape, then rename to your model. **Start with recipes 1 and 2**: they show what every good screen does — compose a record's facts, name things for where they sit, keep the normal quiet and make the exception loud.

## 1. Record page — a composed record

<!-- recipe: views/patron_record.springml -->
```jsx
{/* @catalog spring-ui */}
<RecordPage type="patron" icon="id-card">
  <Query id="out" type="loan" sort="due_on" page={0} search={false} where={l => l.patron == record && l.state == "out"}>
    <Flag condition="is_overdue" />
  </Query>
  <Query id="holds" type="hold" sort="ready_at" page={0} search={false} where={h => h.patron == record} />
  <Query id="fines" type="fine" sort="due_on" page={0} search={false} where={f => f.patron == record}>
    <Flag condition="is_overdue" />
  </Query>
  <Aggregate id="billed" of={fines} op="sum" property="amount_cents" where={f => f.status != "waived"} />
  <Aggregate id="paid" of={fines} op="sum" property="paid_cents" where={f => f.status != "waived"} />
  <Aggregate id="owed" of={fines} op="sum" property="open_cents" />
  <Meta>
    <Fact field="status" />
    <Fact field="card_no" sub={record.home_branch} />
    <Fact field="expires_on" />
    <Fact label="Owed" value={owed.value} />
  </Meta>
  <Action run="patron_activate" role="primary" />
  <Action run="patron_renew_card" />
  <Action run="patron_suspend" />
  <Action run="patron_reinstate" />
  <Action run="patron_close" />
  <Split>
    <Section title="Patron">
      <Facts>
        <Fact field="name" sub={[record.email, record.phone]} />
        <Fact label="Member" span={["joined_on", "expires_on"]} />
        <Fact field="id_scan" />
      </Facts>
    </Section>
    <Section title="Loans">
      <Breakdown of={record}>
        <Line value={billed.value} label="Fines" />
        <Line value={paid.value} kind="minus" label="Paid" />
        <Line value={owed.value} kind="due" label="Still owed" />
      </Breakdown>
      <Meter value={record.loans_out} of={record.loan_limit} />
    </Section>
  </Split>
  <Section title="On loan">
    <List of={out}>
      {l => <Item title={l.item} sub={[l.format, l.due_on]} />}
      <Action run="loan_renew" />
      <Empty title="Nothing on loan">This patron has nothing out.</Empty>
    </List>
  </Section>
  <Section title="Holds">
    <List of={holds}>
      {h => <Item title={h.item} sub={[h.shelf, h.pickup_by]} />}
      <Empty title="No holds">Nothing is waiting for this patron.</Empty>
    </List>
  </Section>
  <Section title="Fines" audience={["librarians"]}>
    <List of={fines}>
      {f => <Item title={f.reason} sub={[f.due_on, f.status]} value={f.open_cents} />}
      <Action run="fine_record_payment" />
      <Empty title="No fines">This patron owes nothing.</Empty>
    </List>
  </Section>
  <Section title="History" audience={["librarians"]}>
    <History of={record} />
  </Section>
</RecordPage>
```

What the owner gets (the page as text; the refused verbs and the librarians' Fines and History sections left out):

```text
# Lena Hofer
Status: Active
Card: C-10442 (Central Library)
Expires: 20 Dec
Owed: €14.50
> Renew card
> Suspend
> Close membership
## Patron
Name: Lena Hofer (lena.hofer@example.org · +43 660 120 4455)
Member: 12 Mar 2024–20 Dec
ID scan: Lena Hofer ID.pdf
## Loans
Fines  €19.50
Paid  €5.00
Still owed  €14.50
3 of 10
## On loan
- The Long Way Home · Book · Sun, 22 Nov
    > Renew
- Night Trains · Audiobook · Fri, 4 Dec
    > Renew
## Holds
- The Long Way Home · H-03 · Fri, 27 Nov, 07:00
- Night Trains · H-11 · Sun, 29 Nov, 18:00
- Small Kitchens · H-07 · Sun, 29 Nov, 18:00
```

A volunteer gets the same page with "Hidden for your role" for the email, phone and money, and no ID scan.

- **`Meta` is the header strip**: the status first (its chip beside the title), then ≤ 5 facts the reader needs first, each with the one sub line that completes it (the card and its home branch). A value that does not name itself (money, a count) takes its label.
- **A named condition of the record is a flag in the header.** A `<Flag condition="is_overdue" />` directly in the `RecordPage` (not in a Query) flags the page record itself: when the condition holds, the header shows one line of flags under the title with the words the model gave it ("Overdue · 2 d"), in its level's colour; when it does not, nothing. A condition that reads a value the reader may not see is left out, never a "hidden" flag. A `Callout` written with the same `condition` shows only while the record's condition holds, in the tone of its level, with your words as its text or, with none, the condition's label and amount. Use the header flag for what a reader must notice at once and a Callout for what needs a sentence.
- **Write every verb the record has; never a `when` for legality.** The platform offers each where the record's machine and the reader's rights allow it and says why the others are not; `role="primary"` marks the forward one. The header shows the primary as its button and puts the others in ⋯ in written order; `role="secondary"` keeps one (at most two) beside it as a button, the one the reader reaches for most. A `when` is for the page's own conditions (a day that is not today).
- **A composed sub line is a list of parts: `sub={[record.email, record.phone]}`.** Each part keeps its own form, they share one line with a quiet dot between, and **a blank part drops with its separator** (a patron with no phone reads "lena.hofer@example.org", never "lena.hofer@example.org · —"). Pick parts that name themselves; a bare number needs its label, so give it a Fact. A `<Text>` sentence is for your own words around values ("in <Value/> days"): there a blank keeps its dash.
- **Regions:** the overview (everything up to the first `Section`, `Split` or `Columns`) is the record at a glance; each top-level Section is a region of its own, label-left `Facts` inside; a `Split` stacks on a phone. **A file is a Fact** (`field="id_scan"`), never a one-row Table.
- **Name rows for the page they sit on.** Every loan here is this patron's, so a loan is its item (`<Item title={l.item}>`), a fine its reason: never the patron's name again on every row.
- **Tabs come on their own:** more than three titled Sections after the overview draw as tabs, one per Section, named by its title and counted by its first collection, the chosen one in the URL. Write no `Tabs` for this; a top-level `Tabs` of your own replaces it.
- **Per-group sections:** `audience={["librarians"]}` on a `Section` or `Tab` places it for those groups only; anyone else is not served it (no title, no "hidden" line). It is placement, not security: the policy decides single values (a volunteer's policy hides a patron's email, phone and ID scan, and every fine), and a `Fact`, `Section` or `Tab` with nothing the reader may read is left out, a `Split` left with one side gives it the full width.

```jsx
{/* @catalog spring-ui */}
<RecordPage type="fine" title={[record.reason]}>
  <Flag condition="is_overdue" />
  <Callout condition="is_overdue">Settle this before the next loan.</Callout>
  <Facts><Fact field="reason" /><Fact field="open_cents" /></Facts>
</RecordPage>
```

## 2. Study rooms — the normal quiet, the exception loud

<!-- recipe: views/room_days.springml -->
```jsx
{/* @catalog spring-ui */}
<Page icon="calendar-check" title="Study rooms" subtitle="Seats per room and day, against the room's seats and the seats patrons asked for" lead={roomDays}>
  <Query id="roomDays" type="room_day" sort={["date", "room"]} page={0} search={false} nameBy="date">
    <Flag condition="is_overbooked" />
  </Query>
  <Grid>
    <Stat label="Days overbooked" value={roomDays.count(d => d.is_overbooked)} context={<Text>of <Value of={roomDays.count()} /> room days</Text>} />
  </Grid>
  <Section title="Days">
    <Table of={roomDays}>
      <Column field="date" sub={d => d.room} />
      <Column label="Booked of seats" value={d => <Ratio value={d.booked} of={d.seats} />} />
      <Column label="Seats of wanted" value={d => <Ratio value={d.seats} need={d.wanted} />} />
      <Empty title="No room days">A study room gets a row per day once patrons book it.</Empty>
    </Table>
  </Section>
</Page>
```

- **Attention comes from the model, never from the page.** "More booked than seats" is the named condition `is_overbooked` on `room_day` (a derived boolean the type gives a `critical` level); `<Flag condition="is_overbooked" />` on the Query marks that row, names it in the header (`nameBy="date"`) and gives the Stat its number (`roomDays.count(d => d.is_overbooked)`). The page never re-derives the rule.
- **`need` and `of` say what the second number means, and that decides the tone.** `need` is a requirement or a target: below it is **short** (critical, "short 2"), at it is **met** (quiet: no bar, no colour), above it is over (a warning). `of` is a capacity: at or under it the pair is plain; past it is **over** (critical, "over by 2"). Choose by what the number is, not by the colour you want; there is no prop for the tone.
- **The normal is quiet.** Two of three days draw as plain numbers ("6 of 8", "3 of 8"); only Saturday carries colour and a word ("10 of 8 · over by 2", the row marked overbooked), so the eye finds it.
- **A type with no title is named by its first Column.** `room_day` has no title (a day of a room is not a thing with a name), so its first Column, the date, heads each row and is its link; give a titled type's rows an `Item` title or a first Column for the page they sit on (recipe 1).
- **A sub line instead of another column.** `sub={d => d.room}` puts the room under the date: one column the reader scans, not two.

**Not this:** a `when` on a verb to hide it where it is not legal (the platform does that, and says why); a Column that repeats the row's title or the page's record; a `<Text>` with literal separators to join values (`<Value of={a}/> · <Value of={b}/>`: a blank draws "— · —"; write `sub={[a, b]}`); `of` for a target (it never says short).

## 3. Today queue — the hold shelf

<!-- recipe: views/holds_today.springml -->
```jsx
{/* @catalog spring-ui */}
<Page icon="library" title="Hold shelf" subtitle="Holds ready for pick-up · oldest pick-up time first" lead={holds} params={{date: "date"}}>
  <State id="day" type="date" initial={param("date") ?? today()} />
  <Query id="holds" type="hold" sort="pickup_by" page={50} search={false} nameBy="shelf" where={h => h.ready_on == day} current={h => h.state == "ready"}>
    <Flag condition="is_expiring" />
  </Query>
  <Aggregate id="byState" type="hold" op="count" by="state" where={h => h.ready_on == day} />
  <Aggregate id="perDay" type="hold" op="count" by="ready_on" />
  <PeriodNav state={day} span="day" counts={perDay} skip="empty" />
  <Grid>
    <Stat label="Handed over" value={byState.at("collected")} />
    <Stat label="On the shelf"><Ratio value={byState.at("ready")} of={holds.count()} /></Stat>
    <Stat label="Waiting" value={byState.at("ready")} context={<Text><Value of={holds.count(h => h.is_expiring)} /> past their pick-up time</Text>} />
  </Grid>
  <Section title="Holds">
    <List of={holds}>
      {h => <Item lead={h.pickup_by} title={h.item} eyebrow={h.shelf} sub={[h.patron_name, h.card_no]} detail={[h.phone]} />}
      <Action run="hold_collect" role="primary" when={day == today()} />
      <Action run="hold_extend" when={day == today()} />
      <Empty title="An empty shelf">No hold is ready this day.</Empty>
    </List>
  </Section>
</Page>
```

A **queue**: the List's Query is sorted by time and has `current`, so the first ready hold whose pick-up time has not passed gets "→ Next", the ones past it carry a rail and a filled pill (the `is_expiring` condition: critical, on `pickup_by`), and the header names them in a fixed form ("pick-up time passed: 27 Nov, 07:00 H-03 · next: 29 Nov, 18:00 H-07" — `nameBy="shelf"` picks the hold shelf as the name). A queue's verbs are `job` by default: the first offered verb, "Hand over", is quiet text on every row it is legal for and bordered on the rows past their pick-up time; "Keep two more days" (`is_expiring`'s fix) waits in each row's ⋯; a hold already handed over offers neither (the machine). The Stats read one grouped Aggregate (`byState.at("collected")`) and the Query's own handles (`holds.count()`, `holds.count(h => h.is_expiring)`), all in one read stage. `search={false}`: a short shelf needs no search box. The shelf is one day: the page takes a `date` parameter (`params={{date: "date"}}`, given as `?date=2026-11-26`; a value that is not a date is refused), the State `day` starts at it or today, the Query and the Aggregate keep the holds that became ready on that day (`h.ready_on`, a derived day of the timestamp `ready_at` in the org's zone), and the PeriodNav in the header steps a day back, to today and on by writing `date` — the counts follow. The verbs are offered only on today (`when={day == today()}`).

## 4. Fines — a worklist with presets and totals

<!-- recipe: views/fine_payments.springml -->
```jsx
{/* @catalog spring-ui */}
<Page icon="banknote" title="Fines" subtitle="Late returns, lost items and damage · payments go to the oldest open fine first" lead={fines}>
  <Query id="fines" type="fine" sort="due_on" page={14}>
    <Preset id="due14" label="Due in 14 days" short="Due 14 d" where={f => f.due_on >= today() && f.due_on <= addDays(today(), 14) && f.status in ["open", "part_paid"]} default />
    <Preset id="overdue" label="Overdue" where={f => f.is_overdue} />
    <Preset id="part_paid" label="Part paid" where={f => f.status == "part_paid"} />
    <Preset id="paid" label="Paid" where={f => f.status == "paid"} />
    <Preset id="all" label="All" />
    <Flag condition="is_overdue" />
  </Query>
  <Grid>
    <Stat label="Overdue" value={fines.where(f => f.is_overdue).sum(f => f.open_cents)} context={<Text><Value of={fines.count(f => f.is_overdue)} /> fines</Text>} />
    <Stat label="Open in this view" value={fines.sum(f => f.open_cents)} />
  </Grid>
  <Toolbar>
    <Filters of={fines} />
    <Search of={fines} />
  </Toolbar>
  <Section>
    <Table of={fines} verbs="job">
      <Column field="ref" importance="primary" sub={f => f.card_no} />
      <Column field="patron_name" />
      <Column field="reason" />
      <Column field="due_on" />
      <Column field="amount_cents" total="sum" />
      <Column field="paid_cents" total="sum" />
      <Column field="open_cents" total="sum" importance="secondary" />
      <Column field="status" />
      <Action run="fine_record_payment" />
      <Empty title="No fines yet">Fines appear here as soon as a loan comes back late.</Empty>
    </Table>
  </Section>
</Page>
```

A **Table** because the reader works the columns: sorts, footer totals over every page, presets with live counts. The `due14` preset is the default; the `overdue` one narrows by the named condition (never by re-deriving "due before today and not paid" in the view). `verbs="job"` makes "Record payment" quiet text on every open row, bordered on overdue ones (it is the condition's fix); the header Action is the same action with a fine picker (no fine in the page's scope). Overdue rows carry their mark, and the header counts them ("overdue: 1") as a filter link. A volunteer, whose policy may not read fines, gets the page's one "hidden for your role" line instead of the table.

## 5. A list with bulk verbs and an expandable row

<!-- recipe: views/loan_list.springml -->
```jsx
{/* @catalog spring-ui */}
<Page icon="book-open" title="Loans" lead={loans}>
  <Query id="loans" type="loan" sort="due_on" page={14}>
    <Preset id="out" label="Out" where={l => l.state == "out"} default />
    <Preset id="week" label="Due this week" where={l => l.due_on >= weekOf(today()).start && l.due_on <= weekOf(today()).end} />
    <Preset id="overdue" label="Overdue" where={l => l.is_overdue} />
    <Preset id="all" label="All" />
    <Flag condition="is_overdue" />
  </Query>
  <Action view="patron_new" label="New patron" role="primary" />
  <Toolbar>
    <Filters of={loans} />
    <Search of={loans} />
  </Toolbar>
  <Section>
    <Table of={loans}>
      <Column field="item" importance="primary" />
      <Column field="patron" />
      <Column field="format" />
      <Column field="out_on" />
      <Column field="due_on" />
      <Column field="renewals" />
      <Column field="state" />
      <BulkAction run="loan_renew" />
      <Action run="loan_renew" />
      <Action run="loan_return" />
      <Expand>{l => <Section title="Renewals">
        <Query id="rn" type="renewal" sort="renewed_on" page={0} where={r => r.loan == l} />
        <Table of={rn}>
          <Column field="renewed_on" importance="primary" />
          <Column field="new_due_on" />
          <Empty title="No renewals">This loan was never renewed.</Empty>
        </Table>
      </Section>}</Expand>
      <Empty title="No loans yet">Loans appear here as soon as something is checked out.</Empty>
    </Table>
  </Section>
</Page>
```

Presets for the common slices, Filters and Search in the Toolbar, a page header verb that opens another view (`view="patron_new"` with its `label`). The row verbs are the loan's actions: each row offers only the ones legal from its state, and with no flag naming them as a fix they sit in the row's ⋯ ("Renew" is the overdue loans' fix, so it rests on those rows). `BulkAction` adds a selection column on the rows that may be renewed and runs `actions.invoke_many` over the selection. `Expand` opens a row in place with a Query scoped to that row (`r.loan == l`), read only when it opens.

## 6. Week chart — a day series

<!-- recipe: views/loan_week.springml -->
```jsx
{/* @catalog spring-ui */}
<Page icon="calendar-check" title="Loans this week" subtitle="Items checked out each day, by format" lead={wk}>
  <State id="week" type="week" initial={weekOf(today())} />
  <Query id="wk" type="loan" sort="out_on" page={0} where={l => l.out_on >= week.start && l.out_on <= week.end} />
  <Aggregate id="out" type="loan" op="count" by="out_on" over={days(week)} />
  <Aggregate id="byFormat" type="loan" op="count" by={["format", "out_on"]} over={days(week)} />
  <Aggregate id="due" type="loan" op="count" by="due_on" over={days(week)} />
  <Grid>
    <Stat label="Out today" value={out.at(today())} compare="previous_period" trend={out} />
    <Stat label="Out this week" value={wk.count()} />
    <Stat label="Due today" value={due.at(today())} />
  </Grid>
  <Columns>
    <Section title="Out each day">
      <Chart shows="composition"><Series of={byFormat} label="Loans" /></Chart>
    </Section>
    <Section title="Due back each day">
      <Chart shows="trend"><Series of={due} label="Due" /></Chart>
    </Section>
  </Columns>
  <Section title="Out this week">
    <Table of={wk}>
      <Column field="item" importance="primary" />
      <Column field="patron" />
      <Column field="format" />
      <Column field="out_on" />
      <Column field="due_on" />
      <Column field="state" />
      <Empty title="A quiet week">Nothing went out this week.</Empty>
    </Table>
  </Section>
</Page>
```

A week State drives everything: the Query's narrowing, and every Aggregate's day series (`over={days(week)}`: a cell per day, an empty day 0). `out.at(today())` is today's cell; `compare` and `trend` give the Stat a delta and a sparkline. `byFormat` is keyed by two fields (`by={["format", "out_on"]}`), so the composition chart stacks the formats in each day. `Chart shows=` names the question — composition (stacked by the first key), trend — and the platform picks the marks.

## 7. Files — a record's attachment kinds

<!-- recipe: views/book_files.springml -->
```jsx
{/* @catalog spring-ui */}
<RecordPage type="book">
  <Meta>
    <Fact field="author" />
    <Fact field="year" />
    <Fact field="format" />
  </Meta>
  <Section title="Files">
    <Files of={record} />
  </Section>
</RecordPage>
```

`Files` shows the record's attachment kinds — each a child type declared on the record's type (`attachments.kinds`, see the canonify-objects-actions skill: a to-many link, the child's `file` property, an optional `kind` value, `facts`, `display: "hero"`) — and its own file properties (here the book's `cover`). Each kind's files are tiles, each by its file name (no thumbnail yet), under the reader's policy: a kind whose child type or file column the reader's role may not read is named "hidden for your role", never listed. "Add <kind>" is the child's create, filled with the record and kind, offered only when the reader may run it. After the kinds comes an **Attachments** block with the files attached to the record itself (an import's `files`, an upload to the record) when it has any — on a type with no kinds, always. Its "Add file" uploads to the record itself, held to the type's `attachments` `accept` / `max_bytes`, offered only when the reader may change the record (its `<type>.update` gate, the upload's own check) — no child type needed for one plain file list. `kinds={["catalogue_card"]}` narrows it to those kinds, without Attachments. How files get there (seeding them with `objects.import`, uploading one): the canonify-objects-actions skill's [files](../../canonify-objects-actions/resources/files.md) page.

A `file` property needs no Files: name it in a `Fact`, `Meta`, `Column` or an `Item` slot and the reader sees the file by its name (no thumbnail or link yet: the kernel has no file read route) — or "hidden for your role" when the policy withholds the property:

<!-- recipe: views/book_documents.springml -->
```jsx
{/* @catalog spring-ui */}
<RecordPage type="book">
  <Query id="docs" type="book_document" sort="-signed_on" page={0} search={false} where={d => d.book == record} />
  <Meta>
    <Fact field="author" />
    <Fact field="cover" />
  </Meta>
  <Section title="Documents">
    <Table of={docs}>
      <Column field="kind" importance="primary" />
      <Column field="signed_on" />
      <Column field="file" />
      <Empty title="No documents">No catalogue card yet.</Empty>
    </Table>
  </Section>
</RecordPage>
```

## 8. Availability over a chosen range — free capacity per row

A derived property cannot read a page State, so a window written into derived SQL ("free seats next 7 days", "delivered in the last 6 weeks") is fixed for every reader. Put the window on the page instead: a range State the reader moves, Aggregates over it, and Columns that read those Aggregates per row.

<!-- recipe: views/room_availability.springml -->
```jsx
{/* @catalog spring-ui */}
<Page icon="door-open" title="Room availability" subtitle="Free seats per study room over the chosen days · from the room-day rows the bookings keep" lead={rooms}>
  <State id="range" type="range" initial={period(today(), addDays(today(), 6))} />
  <Query id="rooms" type="room" sort="seats" page={0} search={false} />
  <Aggregate id="booked" type="room_day" op="sum" property="booked" by={["room", "date"]} over={days(range)} />
  <Aggregate id="bookings" type="room_booking" op="count" by="room" where={b => b.date >= range.start && b.date <= range.end} />
  <Toolbar><Control state={range} label="From" /><PeriodNav state={range} span="fortnight" /></Toolbar>
  <Section title="Rooms">
    <Table of={rooms}>
      <Column field="name" importance="primary" />
      <Column field="seats" />
      <Column label="Booked at peak" value={r => booked.peak(r)} sub={r => booked.peakAt(r)} />
      <Column label="Free every day" value={r => booked.free(r, r.seats)} />
      <Column label="Bookings" value={r => bookings.at(r)} />
      <Empty title="No rooms">Add study rooms to plan the week.</Empty>
    </Table>
  </Section>
</Page>
```

- **"On no day may X exceed Y" needs the busiest day, not the range total.** `booked` is a day series keyed by the row and the day (`by={["room", "date"]}` over `room_day` and `over={days(range)}`): each room-day row holds what is booked that day. Along that series `booked.peak(r)` is the room's busiest day, `booked.peakAt(r)` the day it falls on, and `booked.free(r, r.seats)` what that day leaves free — never below 0: an overbooked room reads **0 · short 2 on Sat 28 Nov** in critical ink and its row wears the rail, so it stands out without a negative number.
- **A capacity that differs by day — `a.free(k, capacity, extra)`.** When what is available on a day is the room's own seats plus what is brought in that day, a scalar capacity reports shortages that are not ones. Give the extra its own day series keyed like the bookings and pass it as the third argument: each day's capacity is `capacity` plus that day's `extra` cell, and the value is the least any day leaves free, short on the tightest day:

Add `<Aggregate id="extra" type="room_day" op="sum" property="extra_seats" by={["room", "date"]} over={days(range)} />` beside `booked` and write the Column `value={r => booked.free(r, r.seats, extra)}`.

  Two chairs brought in for Sat 28 Nov then cover the 10 booked of Study room A's 8 seats: the column reads 0, not "0 short 2". `extra` must be a count or sum Aggregate keyed like `booked` (the row, then the day) in the same unit (`aggregate_peak` otherwise). A handler's `peak_day` reads one type's busiest day, not capacity less bookings per day: a booking guard that must agree with this column reads bookings and extras with `per_day` and checks each day of the request in a `for_each` over the bookings' days (canonify-objects-actions, actions-handler-patterns: each day of a range).
- **A range count is an Aggregate keyed by the row whose `where` reads the State** (`bookings`), read with `.at(r)`.
- **The reader moves the window**: `Control` types its first day, `PeriodNav` steps it by its length; every column re-reads. A measure the reader's policy withholds is withheld in every cell, never 0.
- **The guard says the same day.** An action that books capacity checks the busiest day of the request and names that day in its refusal ("Overbooked on 28 Nov: Study room A has 8 seats, 8 already booked, 2 asked."); the action's precondition is the model's (canonify-objects-actions).

## 9. Par vs stock over a chosen range

The same move on a record page: the record's lines, and per line what the transfers in the chosen range needed and brought.

```jsx
{/* @catalog spring-ui */}
<RecordPage type="branch">
  <State id="range" type="range" initial={period(addDays(today(), -42), addDays(today(), -1))} />
  <Query id="lines" type="branch_stock" sort="book" page={0} where={s => s.branch == record} />
  <Aggregate id="needed" type="transfer" op="sum" property="needed" by="book" where={t => t.branch == record && t.sent_on >= range.start && t.sent_on <= range.end} />
  <Aggregate id="delivered" type="transfer" op="sum" property="delivered" by="book" where={t => t.branch == record && t.sent_on >= range.start && t.sent_on <= range.end} />
  <Section title="Par vs stock">
    <Toolbar><Control state={range} label="From" /><PeriodNav state={range} span="fortnight" /></Toolbar>
    <Table of={lines}>
      <Column field="book" importance="primary" />
      <Column field="par" />
      <Column field="on_shelf" />
      <Column label="Needed" value={s => needed.at(s.book)} />
      <Column label="Delivered" value={s => delivered.at(s.book)} />
      <Column label="Short" value={s => needed.at(s.book) - delivered.at(s.book)} />
    </Table>
  </Section>
</RecordPage>
```

The window is the page's (`period(addDays(today(), -42), addDays(today(), -1))` by default, six weeks back); the per-line facts that do not depend on it (`par`, `on_shelf`) stay fields of the line. A transfer's `needed` and `delivered` are its own fields, and so is the day it was sent (`sent_on`): the predicate narrows by an own field, never through a link.

## 10. Cover — a need against an actual, per row and day

This lesson has no natural case in the library, so it is a community sports club's: its teams and their match days.

<!-- recipe: views/team_cover.springml -->
```jsx
{/* @catalog spring-ui */}
<Page title="Match days" subtitle="Players available of the squad each team needs, per match day" lead={cover}>
  <State id="week" type="week" initial={weekOf(today())} />
  <Query id="teams" type="team" sort="name" page={0} search={false} />
  <Query id="cover" type="team_day" sort={["date", "team"]} page={0} search={false} where={d => d.date >= week.start && d.date <= week.end} />
  <Aggregate id="avail" type="team_day" op="sum" property="available" where={d => d.date >= week.start && d.date <= week.end} />
  <Aggregate id="need" type="team_day" op="sum" property="needed" where={d => d.date >= week.start && d.date <= week.end} />
  <Toolbar><PeriodNav state={week} span="week" /></Toolbar>
  <Grid>
    <Stat label="Players available"><Ratio value={avail.value} need={need.value} /></Stat>
    <Stat label="Teams" value={teams.count()} />
  </Grid>
  <Section title="Cover">
    <Table of={cover}>
      <Column field="date" importance="primary" />
      <Column field="team" />
      <Column label="Players" value={d => <Ratio value={d.available} need={d.needed} />} />
      <Empty title="No match days">Add the week's match days to plan the cover.</Empty>
    </Table>
  </Section>
</Page>
```

A **need against an actual** on every row: `<Ratio value={…} need={…} />` reads "available of needed", and its tone comes from its two numbers — below its need it is **short** (critical "short 2"), at it **met** (quiet), above it **over** (warn "over by 1"). Model a need as rows that carry it (one `team_day` row per team and match day with its `available` and `needed`), never as one column per team or day: the week is then a Query of those rows (`where` reads the week State), one row per team and day, and the totals are Aggregates over the same window. `of=` is the other whole — a capacity: over it is critical, under it is neutral (`<Ratio value={booked} of={seats} />`). The same Ratio draws the same tone in a `Stat`, a `Fact` value or a Column `value={r => <Ratio … />}`. A Column whose `sub` is a number read from a field writes it with that field's label ("Renewals 2"). A grid of teams by day, where the place of a mark is the data, is a Region ([regions.md](regions.md)).
