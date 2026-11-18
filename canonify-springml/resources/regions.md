# Regions — when the layout is the data

Most screens are documents: a record, a list, a worklist, a form, a dashboard. Write them with the Spring UI names ([components.md](components.md)); any legal page of them is a good page. Some screens are programs: **where a mark sits is the data**. A week of bookings by room, a schedule of shifts by person and hour, a board whose cards hold nested marks, a timeline that mixes two record types. For those, the page declares the data and **your code draws one region** of it. A Region draws; it decides nothing: who may read a row, which values are withheld and which verbs are legal are decided on the server before your code runs, as for any page.

| The screen | Write |
|---|---|
| A record, a list or worklist, a form or wizard, KPIs and charts | Spring UI names — never a Region |
| A calendar, a schedule, a resource × time grid, a board of nested marks, a timeline of several record types | a `Region` over Queries the page declares |

## The element

```jsx
<Page title="Room bookings" lead={bookings}>
  <State id="week" type="week" initial={weekOf(today())} />
  <Query id="bookings" type="room_booking" sort={["date", "starts_at"]} page={0} where={b => b.date >= week.start && b.date <= week.end} mine={b => b.patron_member == me()} />
  <Query id="rooms" type="room" sort="name" page={0} search={false} />
  <Toolbar><PeriodNav state={week} span="week" /></Toolbar>
  <Region code="week" data={{bookings: bookings, rooms: rooms}} calendar={opening_hours} range={week}>
    <Action run="room_booking_cancel" />
  </Region>
</Page>
```

- `code` (a plain name, `[a-z][a-z0-9_-]*`) names the module `views/<view>/<code>.region.js` in the bundle (here `views/room_week/week.region.js`), applied with the bundle (`canon catalog apply`) and versioned with it.
- `data={{name: q}}` hands the region the Queries and Aggregates it draws, each under the name your code reads (`model.data.bookings`; a name is `[a-z][a-zA-Z0-9_]*`). A personal or secret property (a phone, an email) reaches it only when the page names it, `fields={{bookings: "phone, email"}}`: masked per reader, its read logged; every other property comes without naming.
- `calendar={opening_hours}` (a CX calendar function of the catalog, its bare name in braces, never a string) with `range={week}` (a date, week, range or month State) gives each day of the range its open hours.
- **Verbs are the Region's `Action` children**, and only they: an action on an entry's type is a row verb (on each row it is legal for), a create of an entry's type an entry verb (`model.data.bookings.verbs`). No Action child: no verbs. A `Group` child groups an entry: `<Region data={{loans: ln}}><Group data="loans" by="copy.branch" /></Region>` gives `model.data.loans.groups`, each `{ key, rows (indexes into rows), collapsed, load?, verbs? }`.
- `mine={r => …}` on a Query marks the reader's own rows (`row.mine`, present only when true), computed in the reader's own read; the region never gets the reader's id. The rest of the page is ordinary SpringML around the Region (a `Toolbar` with `PeriodNav`, Sections, Stats, header Actions).

Refused at compile: `page.bind.region_code` (a `code` that is not a plain name), `page.bind.region_data` (an entry that is not a Query or Aggregate of the page, or a bad name), `page.bind.region_verb` (an Action child that is neither a row verb nor a create of an entry's type). At read, a module the bundle never applied draws as a fault (`page.region.unapplied`): apply the bundle.

## The module

The module has one default export, `draw(root, model)`: the checks and the author loop call it offline. In the page it draws through `platform.onModel(m => draw(document.body, m))`, and the host never calls `draw` itself. It imports nothing. The platform injects `platform` first:

| Call | Does |
|---|---|
| `platform.onModel(fn)` | calls `fn(model)` now and each time the page reads again (after an action, a State change). Draw from scratch each time. |
| `platform.open(verb)` | hands a verb of the model to the shell, which always shows its confirm or the verb's form; call it only in a click or key handler (otherwise it is ignored). The server checks it again. |
| `platform.link(row.opens)` | opens a row's record page (only a record a row of the model opens). |
| `platform.setHeight(px?)` | reports the region's height. It is also kept in step on its own. |

A week of study-room bookings by room, the reader's own marked (`mine`):

```js
const el = (tag, cls, text) => Object.assign(document.createElement(tag), { className: cls ?? '', textContent: text ?? '' });
export default function draw(root, model) {
  root.replaceChildren(); const { bookings, rooms } = model.data;
  const days = model.page.days ?? []; // absent with no calendar=; else the days, or a fault value when they could not be read
  for (const d of [bookings, rooms, days]) if (d.k === 'fault') return root.append(el('p', 'muted', d.value.text)); // its words, never "no bookings"
  const grid = el('div', 'week'); Object.assign(grid.style, { display: 'grid', gap: 'var(--space-2)', gridTemplateColumns: `repeat(${days.length}, minmax(0, 1fr))` });
  for (const day of days) {
    const col = el('section', 'card');
    col.append(el('h2', '', day.date.text)); if (day.open.length === 0 && day.closed) col.append(el('p', 'muted', day.closed.text));
    for (const room of rooms.rows) {
      col.append(el('h3', 'muted', room.record.text));
      for (const b of bookings.rows.filter((b) => b.fields.date.value.v === day.date.value.v && b.fields.room.value.ref?.id === room.id)) {
        const card = el('article', b.mine ? 'card mine' : 'card');
        const a = Object.assign(el('a', '', `${b.fields.starts_at.text}–${b.fields.ends_at.text} ${b.record.text}`), { href: '#' });
        a.addEventListener('click', (e) => { e.preventDefault(); platform.link(b.opens); }); card.append(a, el('p', 'muted', b.fields.patron.text)); // the booker's name says whose it is; `mine` only styles
        for (const v of b.verbs.filter((v) => v.state === 'offered')) {
          const btn = Object.assign(el('button', 'button ghost small', v.label), { type: 'button' }); btn.addEventListener('click', () => platform.open(v)); card.append(btn);
        }
        col.append(card);
      }
    }
    grid.append(col);
  }
  root.append(grid);
}
platform.onModel((model) => draw(document.body, model));
```

## What the model holds

`model` is the page's own reader-scoped model of the named data, the same values the platform's own components draw:

- **`model.data[name]`** is one of:
  - `{ k: 'rows', type, noun: {one, other}, rows, total, cut?, more?, verbs, groups? }`: the Query's rows. `verbs` are the entry verbs (a create). `total` is how many match. `cut` is a string: draw it ("Showing 500 of 812"); `more` says more rows follow, how many unknown. So "none" and "not all shown" differ.
  - `{ k: 'fault', value }`: the read did not happen (it failed, the reader may not read it, or a State it needs is not chosen). Draw `value.text` ("Choose a room", "Hidden for your role"). It is never an empty list.
  - `{ k: 'value', value }`: an Aggregate's one figure.
- **A row** has `id`, `record` (its title, written: `{ text, value }`), `status`, `flags`, `opens` (where the record opens), `verbs`, `mine` (only when true), and `fields`: every property of the row's type the reader may read, by name, each written as `{ text, value }`. Read `row.fields.starts_at.text` for the words you draw and `row.fields.starts_at.value` for your logic: a date's `value.v` is `"2026-11-23"`, and a link's value is the linked record (`value.ref` is `{ type, id }`, its `text` the record's title). A property the reader may not read keeps its key, but its value is withheld (`value.t === 'withheld'`): treat it as absent. Never compare or place by it, draw its `text` ("Hidden for your role") where it would sit, and never guess or leave a blank.
- **`model.page`**: `today`, `zone`, `locale` (`en` or `da`), `states` (the page's States as the URL holds them), and `days`: absent when the Region has no `calendar` and `range` (read it as `model.page.days ?? []`), else one per day of the range, in order, `{ date, open: [{ from, to, toMidnight? }], closed? }`. These are the calendar's weekly hours in the org's wall time, closed (with the holiday's title in the words) on each holiday of the calendar the reader may read; a split day has several `open` entries. Shade the time outside `open`, and on a day with no open hours draw its `closed` words. Never compute hours yourself. A range over 366 days, or a failed read (holidays past the row cap), makes `days` a fault value `{ k: 'fault', value }` instead of a list: draw no days, only its words, as for a data entry's fault.
- **`model.words`**: the region's chrome words in the reader's language: `menu`, `today`, `closed`, `withheld`, `failed`, `retry`, `cancel`. Write none of your own for the platform's states.
- **Verbs**: each is `{ name, label, state, reason?, subject?, emphasis, lead }`. Match and mark a verb by its `name` (the declared action name, e.g. `data-verb={v.name}`); `action` is an internal id, never compare it. Offer `state === 'offered'` ones as buttons. A `refused` verb is a line with its `reason`, never a dead button. Never build a verb yourself: `platform.open` takes one from the model, and the shell runs only a verb it sent.

## The look

The region is part of the product, not a page of its own. The shell draws the navigation, the page header, the toolbar and the accent around it, and injects @forspring/ui's stylesheet, tokens and fonts into the region.

| Draw | With |
|---|---|
| a value | its presented text, `fields[p].text` (a fault's `value.text`, `words.*`): the platform formats numbers, money and dates per reader; never format one yourself (`format` refuses `Intl`, `toFixed`, `toLocale…`); whole amounts: `decimals={{<entry>: 0}}` on the `<Region>` |
| a status | the shell's status component, the record page's own markup and look: `<span class="status" data-hue="<hue>">${fields[p].text}</span>` with hue = `fields[p].value.status` (initial, live1–5, sink, hot, warm, cool, good, info, neutral), `.value.band` or a met condition's `.value.level` (critical, warn, good), or the attention `fields[p].tone`; never by matching its words. `var(--tone-<hue>)` is the same colour for any other mark |
| the accent, a link | `var(--primary)` = `var(--accent)` = the shell's accent; a `format="tel"`/`"maps"`/`"mailto"` field carries `fields[p].link = { href, kind: "tel" | "maps" | "mailto" }` beside its text (built by the server; absent when blank or withheld): draw `<a href={fields[p].link.href}>{fields[p].text}</a>`, never build an href yourself |
| layout, controls | ui's classes (`card`, `button`, `button ghost small`, `muted`) and tokens (`var(--space-3)`, `var(--border)`); `<style>` and `style` work, with no URL in them; never a raw colour or a `font-family`. Two families: Archivo for text and labels, Archivo Condensed only for the title, section headings and big figures, Space Mono only for code and ids to quote or copy (`code`, `kbd`); the library's classes set them. A record's id (LN-2041) is mono, small, muted, above the title, never in it, and the title is the name alone ("Night Trains"); the basics check flags the rest, and a coloured bar down a box's left edge |

- At 390 px, change the axes, never squeeze them: one day at a time, with a day strip, and no sideways scroll. Name what a reader needs (a heading per day, a button's label, a link's text) and reach every verb by keyboard.

## The checks

The frame has no network, storage or navigation and cannot reach the shell's page: drawing is all a region can do. `canon catalog apply` refuses a Region unless an operator has allowed the org for Regions. The author loop and apply run the same checks; a failing one refuses the bundle with its rule and line. The loop also shows a screenshot per role at 1440 and 390: look at each.

| Check | Refuses |
|---|---|
| `colour-literal` | a raw colour (`#hex`, `rgb()`, a named colour) outside a custom property: use ui's tokens |
| `network`, `import`, `default-export` | `fetch`, XHR, WebSocket, EventSource, `sendBeacon`, `new Image`, a URL in CSS or a tag, any `import`: the model is all the data there is; a module without exactly one default export |
| `format` | `Intl.*`, `.toFixed(`, `toLocale…(`: draw the presented text |
| `rtc`, `eval`, `frame`, `subframe`, `storage` | WebRTC or media names; `eval` and `new Function`; `window.parent` / `top` except `parent.postMessage`; a nested frame, object or embed, `frames[`, `.contentWindow`; `localStorage`, `sessionStorage`, `indexedDB`, `document.cookie` (keep state in the page's States) |
| `visibility` | a value in the region's DOM that the reader's model does not hold: draw only what the model gives |
| `overflow-390`, `basics` | a region wider than the window at 390 px; the shell's basics rules (centred labels, spacing, nothing cut, no raw ids) |
