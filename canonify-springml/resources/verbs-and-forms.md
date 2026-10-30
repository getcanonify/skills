# Verbs and forms

A page never writes. Every change is a governed invoke of one of the model's **actions**, and the model owns everything about it — its inputs and their labels, which inputs are required and when, its steps, its preview, its guards and their refusal texts, its submit buttons, what it is called on a row. The page only says **where** a verb sits, and the platform builds the rest: the generated dialog, scope fill, legality, Undo or confirm, refusals, focus. Model-side keys below are declared on the action (its `presentation`, its handler, `offeredWhen`) as the `canonify-objects-actions` skill describes; every page here compiles against the coach-hire model of [components.md](components.md).

## Action — the one verb element

`<Action run="trip.confirm" />` runs an action through its generated dialog; `<Action view="Trip.new" label="New enquiry" />` opens a view (`params` passes its parameters). A `run` verb takes **no** `label` (the model's `label` and `shortLabel` are used: `action_label`); a `view` verb needs one.

**Placement says what the verb acts on** (anywhere else is refused):

| Written in | It is | Subject |
|---|---|---|
| `Page` / `RecordPage` | a header verb (≤ 3 offered together) | none, or `record` |
| `Toolbar` | a toolbar verb | page scope (in a Section: its `of`) |
| `Section` | a section header verb | the Section's `of` |
| `Table` / `List` / `Board` | a row or card verb | the row (or its unique to-one link of the subject type) |
| `Group` | a group verb | the group's key record |
| `Empty` | the call to action | page scope |
| `Cell` | a cell popover verb (≤ 3) | the popover record, else the cell |
| `Timeline` / `DayGrid` | the item's ⋯ menu | the item |

**Scope fill.** Every input of type `ref T` takes the **nearest single T in scope**: the popover record, a Matrix cell's own record (its `cells` record — a cell with none offers no verb on it), the row, the row's unique to-one link of type T, the Matrix row record, the Group key, the Section's `of`, `record`, ref States, Records. An enum input takes the row's field of the same enum; a date input the Matrix column's day (when the action has exactly one date input); a lane input the Board lane or DayGrid slot. Two different T at the same level is refused (`fill_tie`) — say which with `with={{input: value}}` (or a row function `with={t => ({vehicle_class: t.vehicle_class})}`; `null` clears a fill). An input declared `sets: <field>` is never filled from a value equal to the subject's current value of that field (it would be a no-op): it is left empty and the readback says why. A verb whose subject nothing in scope supplies gets a subject picker as its dialog's first field.

```jsx
<Page title="Two trips" params={{a: "trip", b: "trip"}}>
  <Record id="ta" type="trip" key={param("a")} />
  <Record id="tb" type="trip" key={param("b")} />
  <Action run="trip.confirm" with={{trip_id: ta}} />
</Page>
```

**Legality is the model's.** A verb is rendered only where it is legal: a state-machine transition from the row's state, an action whose `offeredWhen` condition holds on the row (`"offeredWhen": "is_today"`, `"!is_cancelled"` or `{"status": ["open"]}` for an action on a type without a machine), and only for a reader granted it. You write every verb the screen may need; each row shows the ones it may run. `role="primary"` may sit on several verbs that exclude each other by legality (the first offered one is primary). A v1 `presentation.showWhen` (or a type's `display.actions.<action>.showWhen`) is not read: a page that runs an action offered only by one is refused (`show_when`) — declare `offeredWhen` on the action instead.

**A parent's verb on child rows.** A row verb whose subject is a link of the row — `trip.complete` on a departure row, its `trip_id` filled from the row's `trip` — is decided per row by that trip: its state, the reader's grants and its preCheck guards. Its legality is the trip's, so every coach of a trip under way would offer it; say which rows do with a **row condition**, a row function on `when`:

```jsx
<Page title="Coaches today" lead={deps}>
  <Query id="deps" type="departure" sort="report_at" where={d => d.report_at == today()} />
  <Section title="Coaches">
    <List of={deps}>
      {d => <Item lead={d.report_at} title={d.title} sub={[d.coach]} />}
      <Action run="departure.check_in" role="primary" />
      <Action run="trip.complete" when={d => d.state == "back"} />
    </List>
  </Section>
</Page>
```

"Complete trip" is offered only on a coach that is back, and only while its trip is under way. A row condition is placement, not security: the action's own guards still refuse a wrong call. Only a Table, List or Board row verb takes one (`row_when` elsewhere); a header verb's `when` stays an expression.

**At rest — verbs follow the flag.** A collection does not show every verb on every row:

- `verbs="flag"` (the default): a verb shows at rest only on rows whose flag it fixes — the condition's `fix` names it — as a bordered button; elsewhere it is in the row's ⋯ menu.
- `verbs="job"` (the default for a queue: a Query with `current`, or a List with a `Check`): the first offered verb is the page's per-row job, shown on every eligible row as quiet text with the action's `shortLabel` ("Record", "Check out"), bordered on flagged rows.
- `verbs="menu"`: every verb in ⋯. `rest={false}` on an Action keeps it out of the resting slot. A Preset may switch the mode while it is active (`<Preset … verbs="job" />`).
- A verb the model marks `preCheck` (a refuse guard over the subject row, `"preCheck": true` on its `assert` step) is evaluated per row in the same read: a row where it would refuse never shows it at rest, and where it shows (a menu, a header) it is disabled with the guard's own text.
- Verbs of one model `group` render as one menu; a `destructive` verb is red text, never filled, in ⋯ on rows; a reversible verb gets Undo, an irreversible one a confirm.

```jsx
<Page title="Payments" lead={pay}>
  <Query id="pay" type="charge" sort="due_on"><Flag condition="is_overdue" /></Query>
  <Action run="charge.record_payment" role="primary" />
  <Section>
    <Table of={pay}>
      <Column field="trip_ref" importance="primary" />
      <Column field="due_on" />
      <Column field="open_cents" total="sum" />
      <Action run="charge.record_payment" />
    </Table>
  </Section>
</Page>
```

Here "Record payment" is a bordered button on overdue rows only (it is `is_overdue`'s fix) and in ⋯ elsewhere; the header verb asks for the charge in its dialog. With `verbs="job"` on the Table it would be quiet "Record" text on every open row. The readback prints the resting verb of each collection and each fill: `charge.record_payment (row): charge_id ← row`.

**One way per action on a page.** The same action as a row verb and a header verb, or as a BulkAction and a row verb, is fine. A `Form` and an `Action` for the same action on one page is refused (`form_twice`): the Form is the screen, or the Action opens the generated dialog — pick one.

## BulkAction and Check

`<BulkAction run="trip.confirm" />` in a Table or List adds a selection column; the action must have a subject (one record: `bulk_subject`). The selected rows run through `actions.invoke_many` — one governed invoke per row (its own guards, transaction and audit row), ≤ 100 per call — and the result reads "8 done · 2 refused" above the table, one line per refused row with the model's own refusal text; refused rows stay selected. Rows where the action is not offered get no checkbox.

Recipe 3 (`Trip.list`) in [recipes.md](recipes.md) puts a BulkAction beside the same action as a row verb.

`<Check run="departure.check_out" undo="departure.undo_check_out" />` makes a List a checklist: `run` and `undo` are transitions of the row type (refused otherwise: `check_transition`), `done` names the condition that counts as checked (default: `run`'s target state). The header counts "n of m done"; a Check replaces the resting verb, and every Action goes to ⋯.

## Form — the action as the screen

Most verbs need no Form: the generated dialog already holds the inputs in model order and steps, conditional inputs, pickers with their facts, candidates, array lines, the live preview and the refusals. Write a `Form` only when the form **is** the screen: its main content with other content beside it (a builder with its own summary), a preview that needs a layout the dialog does not give (a Breakdown beside the inputs, a Table of a preview array), or a page that chooses the subject before anything else.

- `<Form run="…" with={{…}}>` — `with` presets inputs (`null` leaves one to its model default); the subject fills from scope as for an Action (`record` on a RecordPage, a Section's `of`), else it is a picker.
- `<Field name="…" />` places one input. Its label, help, picker, whether it is required (`required`, `requiredWhen`) and whether it is asked at all (`inputShowWhen`) are the model's — Field takes `name` and nothing else.
- `<Step name="…">` lays out one of the model's named `steps`; its Fields must belong to that step. A Form without Steps renders the model's steps itself; content outside the Steps stays on every step; the platform adds the check-answers step.
- `<Submit mode="…" />` is one of the model's named `submits`, required when the model has more than one. A submit sets inputs (`save` sets `send_quote: false`) and may name **`next`**: an action run on the same record once the first has committed ("Save and send quote" = enquire, then `trip.quote`). The submit with a forward `next` is the primary one.
- `<Preview id="pv" />` puts the action's typed preview outputs in scope: scalars as `pv.total`, an array output as a Table/List `of={pv.items}`, a keyed output for `<Line each={pv.vat_by_rate} />` (the generated dialog draws a keyed output one line per key by itself — what a payment pays on each charge), a line's own value as `line.<property>` inside its Lines (below). Outputs the reader may not read are withheld.

A guard with `severity: "warn"` ("over 40 h this week is overtime", "only 9 h rest since the last shift") lets the call run, and the person reads its text **before** Save: the generated dialog and a Form page with a Preview show each warning the action gives for the current inputs above the Submit, in the guard's own words, as the inputs change — the same live preview that prices the form, nothing saved. An action whose only preview concern is a warning needs no `preview` declared. A refusal is said apart, in the critical tone; a warning never blocks Submit.

**A Form beside its result.** `<Split>` in a Form lays the inputs beside what they produce. The side that holds the Form's parts (`Field`, `Lines`, `Step`, `Submit`, `Candidates`) is sized by its fields (at most about 28rem); the other side — the Breakdown, Table or Stat of the preview — takes the rest of the width, whichever order you write them in. Write the inputs first to put them on the left:

```jsx
<RecordPage type="trip">
  <Form run="trip.add_extras">
    <Preview id="pv" />
    <Split>
      <Section title="Extras">
        <Lines field="extras"><Field name="description" /><Field name="qty" /><Field name="unit_cents" /><Value of={line.net_cents} /></Lines>
        <Submit />
      </Section>
      <Section title="Price">
        <Breakdown of={pv}><Line field="extras_cents" label="Extras" /><Line field="total_cents" kind="total" /></Breakdown>
      </Section>
    </Split>
  </Form>
</RecordPage>
```

A Split whose two sides both hold inputs, or neither does, is main + aside as everywhere else.

A Form whose verb is not offered on its subject draws no inputs; it says why, as the model decides it: the subject's state ("Set the price is not offered while Status is Lost"), the `offeredWhen` condition by its label ("Set the price is offered only on trips not yet confirmed"), or the reader's role. A Form a `preCheck` guard refuses shows the guard's text before its Submit, which sends nothing. A refusal of a submit clears as soon as an input changes.

The [Form example](components.md#form) lays `trip.enquire` out in its two Steps with both Submits; the [Preview example](components.md#preview) shows `trip.confirm`'s computed total before it is confirmed. The model behind `trip.enquire` (in its `presentation`):

The model behind it (in `trip.enquire`'s `presentation`):

```json
{
  "label": "New enquiry",
  "steps": [
    { "name": "trip", "title": "Trip", "inputs": ["hirer_id", "title", "destination", "depart_date", "return_date"] },
    { "name": "group", "title": "Group", "inputs": ["passengers", "wheelchair_users", "access_notes", "vehicle_class"] }
  ],
  "inputShowWhen": { "access_notes": [{ "field": "$input.wheelchair_users", "op": "gt", "value": 0 }] },
  "requiredWhen": { "access_notes": [{ "field": "$input.wheelchair_users", "op": "gt", "value": 0 }] },
  "submits": [
    { "name": "save", "label": "Save enquiry", "set": { "send_quote": false } },
    { "name": "quote", "label": "Save and send quote", "set": { "send_quote": true }, "primary": true, "next": "trip.quote" }
  ]
}
```

### Lines — an array input

`<Lines field="extras">` lays out an array input as a grid — a row per line, a column per sub-input (`<Field>`s of the row's inputs) — with Add and Remove; `<Value of={line.net_cents} />` shows each line's own value beside it, read-only, in a column headed by the value's label. A line's value is a property of the row the action inserts for that line: declare it in the action's `presentation.preview.rows` — `"rows": { "extras": ["net_cents"] }` when the handler's `for_each` over `$input.extras` inserts one row per line — and the Form's Preview answers it per line as the line changes (a line still being filled in, or one the action refuses, shows none). Without that declaration `line.net_cents` is refused at compile with the fix; a dotted output name (`"outputs": { "extras.net_cents": … }`) is refused at declare, as a line's value is the row it inserts, not an output. A generated dialog draws every array input the same grid by itself. Each column is headed by the row field's label: declare it as `"inputLabels": { "lines.qty": "Credit qty" }` (`<array input>.<row field>`; without one the column is the field's name in words). The model may **seed** the opening lines (`presentation.seedFrom`): one per child of the subject (`"from": "$subject.<to-many link>"`), or one per row of a preview output (`"from": "$preview.<array output>"` — e.g. proposed dates each with its outcome ok / skipped / conflict, so the person moves or skips the conflicts before saving), with `map` taking each line's fields (map a child property to a row field to suggest its value: `"qty": "creditable_qty"`), `label` naming the line, and — for a `$subject` seed — `facts` naming child properties each line shows read-only beside its inputs (`"facts": ["delivered_qty", "credited_qty"]`). That is how multi-row work is one dialog: a return per stop (one line per item, `on_shelf` / `delivered` / `soiled_in` inputs, `par` and `packed` as facts), issuing stock for every short item, a credit note with delivered and credited columns and the preview's totals under the grid. A line the person adds takes the row fields' `inputDefaults` (`"lines.qty": 0`), and so does every seeded line for a field its `map` does not fill (a literal default: a credit note's lines all open at `0`); `"lines.unit_price": "$row.item_id.list_price"` fills the unit price when the line picks its item (from the record the picker listed, as the reader reads it).

The [Lines example](components.md#lines) is `trip.add_extras` with a Breakdown of its preview beside the lines.

Were the trip to keep its extras as a to-many `extras` link, reopening them as lines would be, in the action's `presentation`:

```json
{ "seedFrom": { "extras": { "from": "$subject.extras", "map": { "description": "description", "qty": "qty", "unit_cents": "unit_cents" }, "label": "description" } } }
```

A `$subject` seed opens the lines on a Form screen, in a header, section or toolbar verb's dialog, and in a row verb's dialog (`<Action run="quote.revise_lines" />` inside a Table, List or Board opens on that row's children). The page reads the children as the person reading it may: a child their policy's row filter hides opens no line, and the lines end with "Other lines" and why (never how many). A column the policy withholds fills no field, and the lines say why. More children than the input's `max` open none, and the lines say so, so the person adds the lines they need. Each line is named by `label`: a child property, or a to-one link of the child (`"label": "item"`), which names the line by the linked record's title as the reader reads it ("Line 2" when they may not read it, never the id). `"fixed": true` keeps the lines to the subject's own children: no Add, no Remove (a return per stop, a credit note over the invoice's lines). `"where"` opens a line only for the children it matches, a filter map over the child's own properties with literal values (an enum's value one of its tokens): a credit note seeded `{ "from": "$subject.lines", "map": { "line_id": "id" }, "fixed": true, "where": { "kind": "item" } }` offers no Credit qty for its "Contract discount 3 %" line, and a child the where leaves out is never counted as one of the "Other lines". A refusal of one line is said on its row and named by its label in the summary; a live preview that refuses lines leaves them out and shows the totals of the rest, saying how many it leaves out — on a Form screen as in a dialog, and for a line a `for_each … "on_item_fail": "refuse"` refuses as for one a plain loop refuses. A Form screen previews what its Submit would send, as the dialog does: a line still missing a required field is left out of the totals, and nothing previews until every required input holds a value. A row verb's dialog reads its `$subject.<field>` defaults and help off the row's record, whether or not a Column shows that field. Map the child's `id` to a `ref` field of the row (`"line_id": "id"`) so the handler knows which child a line changes; a `ref <type>` row field also opens from the key column of the child's to-one link to that type (`"item_id": "item_id"`, the column behind the child's `item` link — map the column, not the link). A row verb's lines are read once for every row on the page, in one read, so the page payload grows with the rows times their children. Keep `map` to the fields the lines edit.

### Candidates — choose with each option's outcome

**Options from a record.** An enum or set<enum> input whose choices are a value on a record the form knows declares it in the model, `presentation.inputOptions` — the contract builder's delivery days are the customer's route days: `"inputOptions": { "delivery_days": "$input.customer_id.route.run_days" }`. `<Field name="delivery_days" />` on a Form page, the generated dialog and a `<Lines>` cell (`"lines.day": …`) then offer only the days that route runs, re-read as the customer changes (every day while none is picked); a day already chosen outside them stays shown, marked invalid, with why ("Not offered: Sat. R4 takes only Mon, Wed and Fri."). The page writes nothing for it, and the handler's `assert` is still the rule Submit meets ([inputOptions](../../canonify-objects-actions/resources/actions-input-governance.md)).

`<Candidates field="driver_id" facts={["licence", "week_driving_min"]} />` shows the records a reference input's picker offers, each by its record's title (as the reader reads it) with its facts written as values (an enum by its label, money in its currency) and the action's own outcome for it — the passing ones first, "Refused (17)" folded with one reason each — from one preview over every candidate (≤ 50). The model declares which inputs have candidates (`"candidates": { "driver_id": { "facts": ["licence", "week_driving_min"] } }` in `presentation`). The generated dialog renders the same by itself; write Candidates only on a Form that is the screen.

See the [Candidates example](components.md#candidates).

## Refusals of verbs and forms

<!-- refused: form_twice -->
```jsx
<Page title="New enquiry">
  <Action run="trip.enquire" />
  <Form run="trip.enquire"><Field name="title" /><Submit mode="save" /></Form>
</Page>
```

<!-- refused: bulk_subject -->
```jsx
<Page title="Trips" lead={t}>
  <Query id="t" type="trip" />
  <Section><Table of={t}><Column field="title" /><BulkAction run="trip.enquire" /></Table></Section>
</Page>
```

Also: `action_label` (a `label` on a `run` verb), `with_key` (a `with` key the action has no input for), `unknown_prop` (a `label` or `when` on a Field), `unknown_input`, `unknown_step`, `unknown_submit` and `submit_mode` (a `<Submit>` without `mode` when the model has several), `unknown_output` (a `pv.` name the action's preview does not declare), `needs_ancestor` (Field, Lines, Step, Submit, Candidates or Preview outside a Form).
