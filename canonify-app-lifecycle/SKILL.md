---
name: canonify-app-lifecycle
description: When to stop adding surface. The five lifecycle rules for a Canonify canon — take one app to a real 1.0 before starting the next, define 1.0 as three to five falsifiable conditions (usually a replay of a closed period, not a live parallel run), park unfinished work out of the nav instead of deleting it, run the retained catalog audit and structural validation before calling anything done, and verify every nav or ViewSpec change in the rendered app. Read when you are handed a brief bigger than one app, or before you declare your second app.
version: 1.0.0
---

# Canonify App Lifecycle — depth-first, and what 1.0 means

[`canonify-app-authoring`](../canonify-app-authoring/SKILL.md) teaches you the
five verbs, and its last section says *"You shipped an app."* After one table.

That is true of the tutorial and false of your canon. The tutorial's shape —
five chapters, one verb each, one domain — is a shape you will unconsciously
repeat across every domain in a large brief, because each chapter is cheap and
finishing is not. Nothing in the other skills tells you when to stop.

This skill is the stopping rules. Five of them, each a default you apply unless
a human overrides it in writing.

| # | Rule |
|---|---|
| 1 | **One app to 1.0 before the next one starts.** |
| 2 | **1.0 is three to five falsifiable conditions**, written before the work. |
| 3 | **Park, don't delete.** Out of the nav, still in the bundle. |
| 4 | **Run the retained catalog audit and structural validation** before you call anything done. |
| 5 | **Verify in the rendered app.** A passing test suite is not a rendered screen. |

> **Where these came from.** A structural sweep of a real production canon: 54
> ObjectTypes, 73 actions, 66 views, 7 apps, 57 tables — and not one app a
> customer could run their week on. Twenty-two views reachable from no nav. A
> view named `_probe.win` wired into a shipping app's navigation. Nine
> ObjectTypes nothing read or wrote. The one domain every other app depended on
> was roughly a tenth built. Every individual declaration was defensible; the
> aggregate was seven prototypes.
>
> Its build plan had also told its agents to work several domains in parallel,
> so some of that breadth was instructed rather than merely undefaulted — a fair
> account of *how it happened*, and not a reason to repeat it. An instruction to
> parallelise is about ordering, never a licence to leave every branch shallow.

---

## Rule 1 — One app to 1.0 before the next one starts

> **Rule.** Finish one app to a stated 1.0 before declaring the first artifact
> of the next one. When a brief names several apps, pick the one the others
> depend on, and build that one first, all the way down.

The failure mode has a name — **parallel prototyping** — and its signature is
*uniform shallow depth across many domains*. Seven apps that each render a list
and a form is not 70% of seven apps; it is 0% of one, because none of them is
usable end to end by anyone.

**It is invisible from inside any single unit of work.** Every ticket you close
adds a genuinely correct artifact; the defect exists only at the level of the
canon, which no single ticket looks at. Doing your current task well will not
catch it. Asking this before you start will:

> *Is there an app in this canon that a real person could do a real day's work
> in, today, without falling back to the system this replaces?*

If the answer is no, that app is your only work. Adding a second domain's tables
while the answer is still no is the failure mode, not progress against it.

**Depth means every layer, not more tables.** An app at 1.0 has its ObjectTypes
refined (links, enums, titles), its lifecycle in a state machine, its writes as
named actions with governance, its views in a nav, and its policies fenced.
Breadth-first canons are shallow in a recognisable way: many ObjectTypes, few
state machines, almost no policies.

**Depending on an unfinished domain is the tell.** If your new app's tables all
carry an FK into a domain that is itself at prototype depth, you are not
building a second app — you are widening the first one's unfinished foundation.

### When a human really did ask for breadth

*Stand up thin surfaces across five domains so stakeholders can see the shape*
is a legitimate ask, and the honest response is not silent compliance. Say what
it costs, in one line — "none of these will be usable end to end until one is
taken to 1.0; which is first?" — then **park** the thin ones (Rule 3) rather
than shipping them into navs.

---

## Rule 2 — 1.0 is three to five falsifiable conditions

> **Rule.** Before the first declaration, write down what would make this app
> done, as three to five conditions a person could **run** and get a yes or a
> no. If a condition cannot be false, it is not a condition.

"Ordering works" cannot be false. It has no subject, no data, no check. Compare:

> **1.0 for the site-ordering app**
> 1. **Who.** The Aarhus site manager, alone, without the old spreadsheet open.
> 2. **On what.** One closed week — week 31 — re-entered from the source orders.
> 3. **The check.** The daily cover count per site for week 31 matches the
>    signed-off week-31 report, every day, every site, exactly.
> 4. **The check.** Every line's price resolves from the effective-dated price
>    table for its own order date, not today's price.
> 5. **The failure path.** A cancelled order after cut-off is refused with the
>    documented error, not silently written.

Every clause names a person, a dataset, and an exact comparison. Each one can
come back *no*.

The shape to copy: **who** starts using it · **on what work** · **the exact
check that proves it** · **at least one failure path**, because an app that only
works on the happy path is not a 1.0.

### The trap: never define 1.0 as a live parallel run

The obvious test — *run the new app alongside the old system for a week and
compare* — sounds rigorous and is almost always unfalsifiable. Users keep
editing the old system: backdated corrections, a cancelled order re-entered, a
price fixed on Thursday for a Monday delivery. The two sides disagree constantly
and **every disagreement is ambiguous**, because there is no authority saying
which side is right. Teams burn weeks reconciling noise and conclude nothing.

> **Replay a closed period instead.** Pick a period that is finished and
> signed-off — last month, a closed quarter, week 31 — where a correct answer
> already exists on paper. Re-enter that period's real inputs and compare
> outputs. Now a mismatch is a defect with an owner, not a difference of
> opinion, and the run terminates.

A replay demands two things of your model, both already required by the
[`canonify-domain-modelling`](../canonify-domain-modelling/SKILL.md)
obligations. **Effective-dated values resolved as of the record's own date** —
replaying July with today's prices reproduces nothing, and resolution keyed to
`$now` instead of the business date is the commonest reason a replay cannot be
made to reconcile. And **integer minor units** — float money produces øre-level
mismatches at exactly the moment reconciliation is your acceptance gate.

Write the conditions where the next agent will find them: the app's own
description, the ticket, the bundle's README. An undocumented 1.0 is
re-litigated by whoever picks the work up next.

---

## Rule 3 — Park, don't delete

> **Rule.** Work that is not in the current slice comes **out of the app nav**
> and **stays in the bundle**. Two separate moves, and you need both.

**Out of the nav**, because an unfinished screen in a navigation is pretending
to be a product. A user who opens it learns the app is unreliable, and every
later agent reads it as a shipped surface and builds on it. Re-declare the app
with `apps.declare` and the item simply is not there; the ViewSpec is untouched.

**Still in the bundle**, because deleting is how canons lose real work.
Declarations are cheap to keep and expensive to reconstruct — the intent, the
column choices and the reasons live in the artifact and nowhere else. Prefer
**rename and refine over delete**: a view renamed from `_probe.win` to
`Deal.win_rate` is a real surface; the same view deleted is an afternoon someone
repeats next quarter. The platform learned this the hard way — one large cleanup
deleted several working subsystems along with the dead ones, and the ones that
mattered are still being rebuilt, months later, from memory.

Parking is three cheap moves: remove the nav entry, keep the ViewSpec declared
and exported, and note in the app's description what is parked and what would
unpark it.

An unfinished surface is still visible in a review when you run
`canon catalog audit <dir>`, but it is not a reason to create a one-off advisory
rule. Clear it by finishing the view and adding it to a nav, or by deleting it
once you are certain it holds nothing.

---

## Rule 4 — Run the retained audit before you call anything done

Run both commands against your own canon before you claim a slice is finished.
The audit is intentionally short: its three retained rules have named
consumers, and the structural pass reports defects that would make apply unsafe.

- [ ] **No shadow entity.** If a declared ObjectType's identity is copied into a
      mapped text column, add a link and derive the display value. →
      `C1-shadow-entity` *(apply-blocking)*
- [ ] **Money uses integer minor units.** A REAL money column is an authoring
      diagnostic until a first-class `money` type exists. → `C10-float-money`
- [ ] **Every write grant is fenced.** Non-owner wildcard writes and unfenced
      hosted-app roles must be rejected. → `C11-wildcard-grant`
- [ ] Run `canon catalog validate <bundle-dir>` and fix every apply-blocking
      diagnostic before applying.
- [ ] Run `canon catalog audit <bundle-dir>` and review its compact
      `N errors · M warnings` summary.

Run the two commands as you go and check the whole canon after **every merge**:
every issue is invisible inside a single change and obvious across the bundle.

Structural validity and coherence are different questions. The compact audit
keeps those questions visible without preserving an unconsumed advisory stream.

---

## Rule 5 — Verify in the rendered app

> **Rule.** A change to a ViewSpec, a nav, an App declaration, or a displayed
> property is not verified until you have **looked at the rendered screen**.
> Tests are necessary and they are not sufficient.

This is not caution, it is documented platform behaviour: a declarable field
must clear several independent allowlists to travel from your declaration to the
browser, and each layer rebuilds the payload from an explicit list of fields, so
an unknown field is **dropped, never rejected**. You get a green suite, an `OK`
envelope, and a screen where the data never arrives. That exact failure has
shipped twice on this platform — code that passed several thousand tests and
changed nothing visible — caught both times only by reading the live surface.

So, for any UI-affecting change:

1. **Prove the slice through the public surface** with a declarative scenario —
   see [`canonify-scenarios`](../canonify-scenarios/SKILL.md). A scenario walks
   your app the way a real caller does, through the same dispatch, with audit,
   outbox, idempotency, governance and the FSM frame all live. That is the
   end-to-end check, and it is declarative like everything else.
2. **Then open the app and look at it.** Confirm the nav shows what you
   declared, the columns are the ones you named, the "Browse data" sidebar
   scopes and nests as intended, and the parked views are absent. The
   agent-browser walkthrough at the end of
   [`canonify-app-authoring`](../canonify-app-authoring/SKILL.md) shows how to
   do this yourself with your own API key.

Assert the **end of the chain**, never the middle: a check against your
declaration proves your declaration; only the rendered app proves the app.

---

## The lifecycle, in order

1. Pick the one app the others depend on.
2. Write its 1.0 as three to five falsifiable conditions — a closed-period
   replay, not a live parallel run.
3. Build it deep: refined ObjectTypes, state machines, named actions, fenced
   policies, views in a nav.
4. Park everything not in the slice — out of the nav, still in the bundle.
5. Run `catalog validate` and the retained `catalog audit`, then fix blocking
   findings.
6. Prove it with a scenario, then look at it in the browser.
7. Run the replay with the person named in condition 1. Only now is it 1.0.
8. **Then** start the next app.

## See also

- [`canonify-app-authoring`](../canonify-app-authoring/SKILL.md) — the five
  verbs this skill sequences. Read it first; come back here when it says
  "You shipped an app".
- [`canonify-domain-modelling`](../canonify-domain-modelling/SKILL.md) — the
  four modelling obligations. Rule 2's closed-period replay is impossible
  without its effective-dating and integer-money obligations.
- [`canonify-scenarios`](../canonify-scenarios/SKILL.md) — the declarative
  end-to-end proof Rule 5 requires.
- [`canonify-viewspecs`](../canonify-viewspecs/SKILL.md) — ViewSpec mechanics and
  rendered-surface verification.
- [`canonify`](../canonify/SKILL.md) — the base manual: declaration verbs, the
  runtime frame, audit and error codes.
