---
name: canonify-app-lifecycle
description: When to stop adding surface. The five lifecycle rules for a Canonify canon — take one app to a real 1.0 before starting the next, define 1.0 as three to five falsifiable conditions (usually a replay of a closed period, not a live parallel run), park unfinished work out of the nav instead of deleting it, run the retained catalog audit and structural validation before calling anything done, and verify every nav or view change in the rendered app. Read when you are handed a brief bigger than one app, or before you declare your second app.
version: 2.0.0
---

# Canonify App Lifecycle — depth-first, and what 1.0 means

The authoring verbs are cheap to repeat, so an agent handed a large brief
repeats them across every domain and ships prototypes, not a product. A real
production canon had 54 ObjectTypes, 73 actions, 66 views and 7 apps — and not
one app a customer could run their week on: 22 views in no nav, a `_probe.win`
view in a shipping nav, nine types nothing read or wrote, and the domain every
app depended on a tenth built. These five rules are the defaults; a human may
override one in writing.

| # | Rule |
|---|---|
| 1 | **One app to 1.0 before the next one starts.** |
| 2 | **1.0 is three to five falsifiable conditions**, written before the work. |
| 3 | **Park, don't delete.** Out of the nav, still in the bundle. |
| 4 | **Run `catalog validate` and `catalog audit`** before you call anything done. |
| 5 | **Verify in the rendered app.** A green suite is not a rendered screen. |

## Rule 1 — One app to 1.0 before the next

When a brief names several apps, build the one the others depend on first, all
the way down. *Uniform shallow depth across many domains* is the failure: seven
apps that each render a list and a form are 0% of one usable app. It is
invisible inside any single ticket, so ask before you start:

> *Is there an app here a real person could do a real day's work in, today,
> without the system it replaces?*

If not, that app is your only work. Depth means every layer — refined
ObjectTypes (links, enums, titles), a state machine for the lifecycle, named
actions with governance, views in a nav, fenced policies. Breadth-first canons
show many ObjectTypes, few state machines, almost no policies. If your new app's
tables all point into an unfinished domain, you are widening that domain's
foundation, not building a second app. An instruction to parallelise is about
ordering, never a licence to leave every branch shallow.

**When a human asks for breadth** ("thin surfaces across five domains"), say the
cost in one line — "none of these is usable end to end until one reaches 1.0;
which is first?" — and park the thin ones (Rule 3) instead of shipping them in navs.

## Rule 2 — 1.0 is three to five falsifiable conditions

Before the first declaration, write conditions a person could **run** and get a
yes or a no. "Ordering works" cannot be false. This can:

> **1.0 for the site-ordering app**
> 1. **Who.** The Aarhus site manager, alone, without the old spreadsheet.
> 2. **On what.** Week 31 — closed — re-entered from the source orders.
> 3. **Check.** Daily cover count per site matches the signed-off week-31 report, every day, every site.
> 4. **Check.** Every line's price resolves from the effective-dated price table on its own order date.
> 5. **Failure path.** An order cancelled after cut-off is refused with the documented error.

The shape: **who** · **on what work** · **the exact check** · **at least one
failure path**.

**Never define 1.0 as a live parallel run.** Users keep editing the old system,
so every disagreement is ambiguous and the comparison never terminates.
**Replay a closed period** instead — finished, signed-off, with a correct answer
on paper — so a mismatch is a defect with an owner. A replay needs values
resolved as of the record's own date (not `$now`) and integer money — both
obligations in [`canonify-domain-modelling`](../canonify-domain-modelling/SKILL.md).
Write the conditions where the next agent finds them: the app's description,
the ticket, the bundle README.

## Rule 3 — Park, don't delete

Work outside the current slice leaves the **nav** and stays in the **bundle**.
An unfinished screen in a nav pretends to be a product, and later agents build on
it. Deleting loses intent that lives nowhere else; prefer rename and refine
(`_probe.win` → `Deal.win_rate`). Parking is three moves: re-declare the app
without the nav entry (the view is untouched), keep the view declared and
exported, and note in the app's description what is parked and what would
unpark it. `canon catalog audit` still shows an unfinished surface; clear it by
finishing it into a nav, or delete it once you are sure it holds nothing.

## Rule 4 — Validate and audit before "done"

Run both against the bundle before claiming a slice, and on the whole canon
after every merge — these defects are invisible inside one change:

- [ ] `canon catalog validate <bundle-dir>` — fix every apply-blocking diagnostic.
- [ ] `canon catalog audit <bundle-dir>` — review the `N errors · M warnings` summary. Its three retained rules:
      `C1-shadow-entity` (identity copied into a text column — link and derive; a name-match warning, so judge it),
      `C10-float-money` (a `REAL` money column — use `type: money` with a currency),
      `C11-wildcard-grant` (a non-owner wildcard write or an unfenced hosted-app role).

Structural validity and coherence are different questions; run both.

## Rule 5 — Verify in the rendered app

A change to a view, nav, App or displayed property is verified only by
looking at the rendered screen. A declared field crosses several allowlists on
its way to the browser, each rebuilding the payload from an explicit field list,
so an unknown field is **dropped, never rejected**: green tests, an `OK`
envelope, and nothing on screen. So:

1. Prove the slice through the public surface with a scenario
   ([`canonify-scenarios`](../canonify-scenarios/SKILL.md)).
2. Open the app and look — nav, columns, the "Browse data" sidebar, parked views
   absent. How an agent does that with its own key: *See it in the app* in
   [`canonify-app-authoring`](../canonify-app-authoring/SKILL.md).

Assert the end of the chain: only the rendered app proves the app.

## The lifecycle, in order

1. Pick the app the others depend on.
2. Write its 1.0 as 3–5 falsifiable conditions — a closed-period replay.
3. Build it deep: ObjectTypes, state machines, actions, policies, views in a nav.
4. Park everything outside the slice.
5. `catalog validate` and `catalog audit`; fix blocking findings.
6. Prove it with a scenario, then look at it in the browser.
7. Run the replay with the person in condition 1. Only now is it 1.0.
8. **Then** start the next app.
