# Verbs and forms

A page never writes. Every change is a governed invoke of one of the model's **actions**, and the model owns everything about it — its inputs and their labels, which inputs are required and when, its steps, its preview, its guards and their refusal texts, its submit buttons, what it is called on a row. The page only says **where** a verb sits, and the platform builds the rest: the generated dialog, scope fill, legality, Undo or confirm, refusals, focus. Model-side keys below are declared on the action (its `presentation`, its handler, `offeredWhen`) as the `canonify-objects-actions` skill describes; every page here compiles against the city library model of [components.md](components.md).

## Action — the one verb element

`<Action run="loan_renew" />` runs an action through its generated dialog; `<Action view="patron_new" label="New patron" />` opens a view (`params` passes its parameters). A `run` verb takes **no** `label` (the model's `label` and `shortLabel` are used: `action_label`); a `view` verb needs one.

**Placement says what the verb acts on** (anywhere else is refused):

| Written in | It is | Subject |
|---|---|---|
| `Page` / `RecordPage` | a header verb (≤ 3 offered together) | none (a create), or the RecordPage's `record` |
| `Toolbar` | a toolbar verb | page scope (in a Section: its `of`) |
| `Section` | a section header verb | the Section's `of` |
| `Table` / `List` | a row verb | the row (or its unique to-one link of the subject type) |
| `Group` (in a Table or a List) | a group verb | the group's key record: the Group's `by` is a link to it (an action on another type is refused, `page.bind.group_subject`) |
| `Empty` | the call to action | page scope |

**The subject comes from where the verb sits**, never from an input: a verb whose action has a subject sits on a row, a Section with `of`, or a RecordPage of that type. On a Page with no such record it is refused (`page.bind.action_subject`), and `with` cannot name the subject (`page.bind.with_key`). **Scope fill.** Every other input of type `ref T` takes the **nearest single T in scope**: the popover record, the row, the row's unique to-one link of type T, the Group key, the Section's `of`, `record`, ref States, Records. An enum input takes the row's field of the same enum. Two different T at the same level is refused (`fill_tie`) — say which with `with={{input: value}}` (or a row function `with={r => ({format: r.format})}`; `null` clears a fill). An input declared `sets: <field>` is never filled from a value equal to the subject's current value of that field (it would be a no-op): it is left empty and the readback says why. 

```jsx
<Page title="Two branches" params={{a: "branch", b: "branch"}}>
  <Record id="ba" type="branch" key={param("a")} />
  <Record id="bb" type="branch" key={param("b")} />
  <Action run="patron_join" with={{home_branch_id: ba}} />
</Page>
```

**Legality is the model's.** A verb is offered where it is legal: a state-machine transition from the row's state, an action whose `offeredWhen` condition holds on the row (`"offeredWhen": "is_today"`, `"!is_cancelled"` or `{"status": ["open"]}` for an action on a type without a machine). **A refused verb is drawn as a plain line with its reason** ("Waive · Only an open fine can be waived.") in a header, a section, a toolbar and a row's menu, never as a disabled or greyed control; a verb the reader's policy denies is not drawn at all. In a header, a verb refused only by the record's state now — its lifecycle state ("Mark lost · Not in a state this action leaves.") or the clock ("Hand over · H-07 is not on the hold shelf yet.") — is not drawn in the header's row: it is a line with its reason in the header's ⋯ menu, so a closed record's header shows only what can be done now. On a row (a Table or List row, and a row's ⋯ menu) such a verb is not drawn at all: a returned loan's row has no "Renew · Not offered here.", its status already says why. The record's state is its lifecycle state, any enum property of it (a `status` with no machine) and the clock. A row verb refused by a guard over the row's other data (a check such as "the order has a delivery date") is not drawn in the row either: a list never repeats a refusal on every line, the rows the guard allows draw the verb, and a column or menu with no drawn verb takes no space. A header, section or toolbar verb refused by a guard keeps its line with the guard's text. A refusal that is still drawn (a header, section, toolbar) fills its guard message's params (`{amount}`) with the record's values when the reader may read every one of them on that page, written as the page writes them (money with its currency, dates in the reader's words); when any one is withheld from the reader, blank, or on a row the reader does not read, it reads "Only for rows marked <condition label>" when the verb is a condition's `fix`, else "Not available on this row". So still write a verb guard's text that reads well without them ("Only an open fine can be waived.", not "Fine {amount} is not open."). You write every verb the screen may need, and never hide one with `when` to stand in for legality. `role="primary"` may sit on several verbs that exclude each other by legality (the first offered one is primary). In a header the primary is the button and every other offered verb is in ⋯, in written order; `role="secondary"` keeps a verb a button beside it (at most two, in written order; a further one stays in ⋯, as does a destructive one, and one the record's state rules out now is still in ⋯ with its reason). `role="secondary"` on a row, card or menu verb has no effect. A v1 `presentation.showWhen` (or a type's `display.actions.<action>.showWhen`) is not read: a page that runs an action offered only by one is refused (`show_when`) — declare `offeredWhen` on the action instead.

**A parent's verb on child rows.** A row verb whose subject is a link of the row — `patron_suspend` on a loan row, its `patron_id` filled from the row's `patron` — is decided per row by that patron: its state, the reader's grants and its preCheck guards. Its legality is the patron's, so every loan of an active patron would offer it; say which rows do with a **row condition**, a row function on `when`:

```jsx
<Page title="Loans out" lead={loans}>
  <Query id="loans" type="loan" sort="due_on" where={l => l.state == "out"} />
  <Section title="Loans">
    <List of={loans}>
      {l => <Item lead={l.due_on} title={l.item} sub={[l.patron]} />}
      <Action run="loan_renew" role="primary" />
      <Action run="patron_suspend" when={l => l.days_late > 14} />
    </List>
  </Section>
</Page>
```

"Suspend" is offered only on a loan more than 14 days late, and only while its patron is active. A row condition is placement, not security: the action's own guards still refuse a wrong call. Only a Table or List row verb takes one (`row_when` elsewhere); a header verb's `when` stays an expression.

**At rest — verbs follow the flag.** A collection does not show every verb on every row:

- `verbs="flag"` (the default): a verb shows at rest only on rows whose flag it fixes — the condition's `fix` names it — as a bordered button; elsewhere it is in the row's ⋯ menu.
- `verbs="job"` (the default for a queue: a Query with `current`, or a List with a `Check`): the first offered verb is the page's per-row job, shown on every eligible row as quiet text with the action's `shortLabel` ("Record", "Hand over"), bordered on flagged rows.
- `verbs="menu"`: every verb in ⋯. `rest={false}` on an Action keeps it out of the resting slot. A Preset may switch the mode while it is active (`<Preset … verbs="job" />`).
- A verb the model marks `preCheck` (a refuse guard over the subject row, `"preCheck": true` on its `assert` step) is evaluated per row in the same read: a row where it would refuse never shows it at rest, and where it shows (a menu, a header) it is a line with the guard's own text.
- Verbs of one model `group` render as one menu; a `destructive` verb is red text, never filled, in ⋯ on rows; a reversible verb gets Undo, an irreversible one a confirm.

```jsx
<Page title="Fines" lead={fs}>
  <Query id="fs" type="fine" sort="due_on"><Flag condition="is_overdue" /></Query>
  <Section>
    <Table of={fs}>
      <Column field="ref" importance="primary" />
      <Column field="due_on" />
      <Column field="open_cents" total="sum" />
      <Action run="fine_record_payment" />
    </Table>
  </Section>
</Page>
```

Here "Record payment" is a bordered button on overdue rows only (it is `is_overdue`'s fix) and in ⋯ elsewhere; written as a header verb it would ask for the fine in its dialog. With `verbs="job"` on the Table it would be quiet "Record payment" text on every open row. The readback prints the resting verb of each collection and each fill: `fine_record_payment (row): fine_id ← row`.

**One way per action on a page.** The same action as a row verb and a header verb, or as a BulkAction and a row verb, is fine. A `Form` and an `Action` for the same action on one page is refused (`form_twice`): the Form is the screen, or the Action opens the generated dialog — pick one.

## BulkAction and Check

`<BulkAction run="loan_renew" />` in a Table or List adds a selection column; the action must have a subject (one record: `bulk_subject`). The selected rows run through `actions.invoke_many` — one governed invoke per row (its own guards, transaction and audit row), ≤ 100 per call — and the result reads "8 done · 2 refused" above the table, one line per refused row with the model's own refusal text; refused rows stay selected. Rows where the action is not offered get no checkbox.

Recipe 5 (`loan_list`) in [recipes.md](recipes.md) puts a BulkAction beside the same action as a row verb.

`<Check run="hold_collect" undo="hold_undo_collect" />` makes a List a checklist: `run` and `undo` are transitions of the row type (refused otherwise: `check_transition`), `done` names the condition that counts as checked (default: `run`'s target state). The header counts "n of m done"; a Check replaces the resting verb, and every Action goes to ⋯.

## Form — the action as the screen

Most verbs need no Form: the generated dialog already holds the inputs in model order and steps, conditional inputs, pickers with their facts, candidates, array lines, the live preview and the refusals. Write a `Form` only when the form **is** the screen: its main content with other content beside it (a builder with its own summary), a preview that needs a layout the dialog does not give (a Breakdown beside the inputs, a Table of a preview array), or a page that chooses the subject before anything else.

- `<Form run="…" with={{…}}>` — `with` presets inputs (`null` leaves one to its model default); the subject fills from scope as for an Action (`record` on a RecordPage, a Section's `of`), else it is a picker.
- `<Field name="…" />` places one input. Its label, help, picker, whether it is required (`required`, `requiredWhen`) and whether it is asked at all (`inputShowWhen`) are the model's — Field takes `name` and nothing else.
- `<Step name="…">` lays out one of the model's named `steps`; its Fields must belong to that step. A Form without Steps renders the model's steps itself; content outside the Steps stays on every step; the platform adds the check-answers step.
- `<Submit mode="…" />` is one of the model's named `submits`, required when the model has more than one. A submit sets inputs (`save` sets `activate: false`) and may name **`next`**: an action run on the same record once the first has committed ("Save and activate card" = join, then `patron_activate`). The submit with a forward `next` is the primary one.
- `<Preview id="pv" />` puts the action's typed preview outputs in scope, each one value (`pv.total`): an output is never an array or a keyed map, so a breakdown by key (net by VAT rate) is a page Aggregate drawn as a Table (`value={r => byRate.at(r)}`); a line's own value as `line.<property>` inside its Lines (below). Outputs the reader may not read are withheld.

A guard with `severity: "warn"` ("this patron already has 9 of 10 loans out", "the room is booked by another group the hour before") lets the call run, and the person reads its text **before** Save: the generated dialog and a Form page with a Preview show each warning the action gives for the current inputs above the Submit, in the guard's own words, as the inputs change — the same live preview that prices the form, nothing saved. An action whose only preview concern is a warning needs no `preview` declared. A refusal is said apart, in the critical tone; a warning never blocks Submit.

**A Form beside its result.** `<Split>` in a Form lays the inputs beside what they produce. The side that holds the Form's parts (`Field`, `Lines`, `Step`, `Submit`, `Candidates`) is sized by its fields (at most about 28rem); the other side — the Breakdown, Table or Stat of the preview — takes the rest of the width, whichever order you write them in. Write the inputs first to put them on the left:

```jsx
<RecordPage type="expense_report">
  <Form run="report_add_items">
    <Preview id="pv" />
    <Split>
      <Section title="New receipts">
        <Lines field="receipts"><Field name="description" /><Field name="qty" /><Field name="unit_cents" /><Value of={line.net_cents} /></Lines>
        <Submit />
      </Section>
      <Section title="Report total">
        <Breakdown of={pv}><Line field="receipts_cents" label="New receipts" /><Line field="total_cents" kind="total" /></Breakdown>
      </Section>
    </Split>
  </Form>
</RecordPage>
```

A Split whose two sides both hold inputs, or neither does, is main + aside as everywhere else.

A Form whose verb is not offered on its subject draws no inputs; it says why, as the model decides it: the subject's state ("Renew is not offered while State is Returned"), the `offeredWhen` condition by its label ("Waive is offered only on fines not yet paid"), or the reader's role. A Form a `preCheck` guard refuses shows the guard's text before its Submit, which sends nothing. A refusal of a submit clears as soon as an input changes.

The [Form example](components.md#form) lays `patron_join` out in its two Steps; the [Preview example](components.md#preview) shows `loan_return`'s late fee before the loan is returned. A model behind such a form, with a guardian asked only for a young patron and two submits (in the action's `presentation`):

```json
{
  "label": "New patron",
  "steps": [
    { "name": "card", "title": "Card", "inputs": ["name", "card_no", "home_branch_id"] },
    { "name": "contact", "title": "Contact", "inputs": ["age", "guardian_name", "email", "phone"] }
  ],
  "inputShowWhen": { "guardian_name": [{ "field": "$input.age", "op": "lt", "value": 16 }] },
  "requiredWhen": { "guardian_name": [{ "field": "$input.age", "op": "lt", "value": 16 }] },
  "submits": [
    { "name": "save", "label": "Save patron", "set": { "activate": false } },
    { "name": "join", "label": "Save and activate card", "set": { "activate": true }, "primary": true, "next": "patron_activate" }
  ]
}
```

### Lines — an array input

`<Lines field="receipts">` lays out an array input as a grid — a row per line, a column per sub-input (`<Field>`s of the row's inputs) — with Add and Remove; `<Value of={line.net_cents} />` shows each line's own value beside it, read-only, in a column headed by the value's label. A line's value is a property of the row the action inserts for that line: declare it in the action's `presentation.preview.rows` — `"rows": { "receipts": ["net_cents"] }` when the handler's `for_each` over `$input.receipts` inserts one row per line — and the Form's Preview answers it per line as the line changes (a line still being filled in, or one the action refuses, shows none). Without that declaration `line.net_cents` is refused at compile with the fix; a dotted output name (`"outputs": { "receipts.net_cents": … }`) is refused at declare, as a line's value is the row it inserts, not an output. A generated dialog draws every array input the same grid by itself. Each column is headed by the row field's label: declare it as `"inputLabels": { "items.qty": "Quantity" }` (`<array input>.<row field>`; without one the column is the field's name in words). The model may **seed** the opening lines (`presentation.seedFrom`): one per child of the subject (`"from": "$subject.<to-many link>"`), or one per row of a preview output (`"from": "$preview.<array output>"` — e.g. proposed dates each with its outcome ok / skipped / conflict, so the person moves or skips the conflicts before saving), with `map` taking each line's fields (map a child property to a row field to suggest its value: `"qty": "open_qty"`), `label` naming the line, and — for a `$subject` seed — `facts` naming child properties each line shows read-only beside its inputs (`"facts": ["spent_cents", "approved_cents"]`). That is how multi-row work is one dialog: checking a report (one line per receipt, `approved` / `rejected` inputs), sending stock to every branch short of its par, a refund with paid and refunded columns and the preview's totals under the grid. A line the person adds takes the row fields' `inputDefaults` (`"lines.qty": 0`), and so does every seeded line for a field its `map` does not fill (a literal default: a refund's lines all open at `0`); `"lines.unit_cents": "$row.item_id.list_price"` fills the unit price when the line picks its item (from the record the picker listed, as the reader reads it).

The [Lines example](components.md#lines) is `report_add_items` with a Breakdown of its preview beside the lines.

Were the report to keep its added receipts as a to-many `receipts` link, reopening them as lines would be, in the action's `presentation`:

```json
{ "seedFrom": { "receipts": { "from": "$subject.receipts", "map": { "description": "description", "qty": "qty", "unit_cents": "unit_cents" }, "label": "description" } } }
```

A `$subject` seed opens the lines on a Form screen, in a header, section or toolbar verb's dialog, and in a row verb's dialog (`<Action run="report_revise_items" />` inside a Table or List opens on that row's children). The page reads the children as the person reading it may: a child their policy's row filter hides opens no line, and the lines end with "Other lines" and why (never how many). A column the policy withholds fills no field, and the lines say why. More children than the input's `max` open none, and the lines say so, so the person adds the lines they need. Each line is named by `label`: a child property, or a to-one link of the child (`"label": "item"`), which names the line by the linked record's title as the reader reads it ("Line 2" when they may not read it, never the id). `"fixed": true` keeps the lines to the subject's own children: no Add, no Remove (checking a report, a refund over the payments of a fine). `"where"` opens a line only for the children it matches, a filter map over the child's own properties with literal values (an enum's value one of its tokens): a report check seeded `{ "from": "$subject.items", "map": { "item_id": "id" }, "fixed": true, "where": { "kind": "receipt" } }` offers no Approved input for its "Per diem" line, and a child the where leaves out is never counted as one of the "Other lines". A refusal of one line is said on its row and named by its label in the summary; a live preview that refuses lines leaves them out and shows the totals of the rest, saying how many it leaves out — on a Form screen as in a dialog, and for a line a `for_each … "on_item_fail": "refuse"` refuses as for one a plain loop refuses. A Form screen previews what its Submit would send, as the dialog does: a line still missing a required field is left out of the totals, and nothing previews until every required input holds a value. A row verb's dialog reads its `$subject.<field>` defaults and help off the row's record, whether or not a Column shows that field. Map the child's `id` to a `ref` field of the row (`"line_id": "id"`) so the handler knows which child a line changes; a `ref <type>` row field also opens from the key column of the child's to-one link to that type (`"item_id": "item_id"`, the column behind the child's `item` link — map the column, not the link). A row verb's lines are read once for every row on the page, in one read, so the page payload grows with the rows times their children. Keep `map` to the fields the lines edit.

### Candidates — choose with each option's outcome

**Options from a record.** An enum or set<enum> input whose choices are a value on a record the form knows declares it in the model, `presentation.inputOptions` — a standing room booking's days are the branch's opening days: `"inputOptions": { "days": "$input.room_id.branch.open_days" }`. `<Field name="days" />` on a Form page, the generated dialog and a `<Lines>` cell (`"lines.day": …`) then offer only the days that branch opens, re-read as the room changes (every day while none is picked); a day already chosen outside them stays shown, marked invalid, with why ("Not offered: Sun. Riverside opens only Mon to Sat."). The page writes nothing for it, and the handler's `assert` is still the rule Submit meets ([inputOptions](../../canonify-objects-actions/resources/actions-input-governance.md)).

`<Candidates field="copy_id" facts={["barcode", "shelf"]} />` shows the records a reference input's picker offers, each by its record's title (as the reader reads it) with its facts written as values (an enum by its label, money in its currency) and the action's own outcome for it — the passing ones first, "Refused (17)" folded with one reason each — from one preview over every candidate (≤ 50). The model declares which inputs have candidates (`"candidates": { "copy_id": { "facts": ["barcode", "shelf"] } }` in `presentation`). The generated dialog renders the same by itself; write Candidates only on a Form that is the screen.

See the [Candidates example](components.md#candidates).

## Refusals of verbs and forms

<!-- refused: page.bind.form_twice(patron_join) -->
```jsx
<Page title="New patron">
  <Action run="patron_join" />
  <Form run="patron_join"><Field name="name" /><Submit /></Form>
</Page>
```

<!-- refused: page.bind.bulk_subject(patron_join) -->
```jsx
<Page title="Patrons" lead={p}>
  <Query id="p" type="patron" />
  <Section><Table of={p}><Column field="name" /><BulkAction run="patron_join" /></Table></Section>
</Page>
```

Also: `action_label` (a `label` on a `run` verb), `with_key` (a `with` key the action has no input for), `unknown_prop` (a `label` or `when` on a Field), `unknown_input`, `unknown_step`, `unknown_submit` and `submit_mode` (a `<Submit>` without `mode` when the model has several), `unknown_output` (a `pv.` name the action's preview does not declare), `needs_ancestor` (Field, Lines, Step, Submit, Candidates or Preview outside a Form).
