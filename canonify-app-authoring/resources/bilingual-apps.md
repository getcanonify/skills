---
title: Go bilingual — locales, @t: message keys, and the i18n bundle
description: Make an app render in two languages — declare supported_locales/default_locale on the App, write display text as @t:<key> message keys, ship one i18n/<locale>.json message file per locale in the bundle, and let catalog.apply fail the build when a translation is missing.
---

# Go bilingual — locales, `@t:` message keys, and the i18n bundle

A Canonify app becomes multilingual **declaratively**, like everything else:
the App declares which locales it supports, every display string is addressed
by a **message key**, and the translations ship as one message file per locale
inside the manifest bundle. The platform then resolves the right language **at
render time** for each viewer — a Danish member sees Danish, an English member
sees English, from the same declared app.

Three rules shape everything in this chapter:

1. **Locale is a presentation concern.** It selects which *declared display
   text* renders — nothing else. It never changes what the API returns.
2. **Identifiers are never localized.** Action names, property names, view
   names, and enum **values** stay English `snake_case` in every locale — they
   are the REST/CLI/MCP contract, byte-identical on every surface. You
   translate what a value *reads as* (`valueLabels`), never the value itself.
   `canon objects list customer --where status=aktiv` must never exist.
3. **Translation drift is a build error.** `catalog.apply` refuses a bundle
   whose declared locales are incomplete — a missing key fails the apply and
   names the key and the locale. You cannot ship a half-translated app.
4. **A multilingual App cannot carry fixed display copy.** When an App declares
   two or more `supported_locales`, every user-facing display string reachable
   from its nav must be an `@t:` key. `catalog validate` reports a
   `elegance_untranslatable_display_literal` warning for each literal; add
   `--strict` when that advisory must make validation fail. `catalog.apply`
   keeps elegance findings advisory and never blocks on this code.

## The three moves

| Move | Where | What |
|---|---|---|
| 1. Declare support | the App (`apps.declare` / `apps/<slug>.json`) | `supported_locales: ["en", "da"]` + `default_locale: "en"` |
| 2. Key your text | every display string you declare | write `@t:<your.key>` instead of literal text |
| 3. Ship the words | the bundle: `i18n/<locale>.json` | one flat `key → text` map per declared locale |

Absent `supported_locales`, or a single entry, the app is **monolingual**:
no locale to honor, no message files required, zero overhead — literal display
strings render as written. This is the default, and the right choice for most
apps. Declare two or more locales only when you mean to maintain both.

## Move 1 — declare `supported_locales` on the App

```json
{
  "slug": "crm",
  "name": "CRM",
  "icon": "briefcase",
  "nav": [{ "view": "Customer.list" }],
  "supported_locales": ["en", "da"],
  "default_locale": "en"
}
```

- `supported_locales` is the closed platform set — currently `en` and `da`.
  An unsupported tag (`"de"`) is rejected at decode.
- `default_locale` must be **named explicitly** (it is never inferred from
  the first array entry) and must be a member of `supported_locales` — the
  apply gate checks the membership (`default_locale_not_supported`).
- Declaring ≥2 locales makes the app **renderable in the viewer's language
  preference** — no per-app wiring needed. The preference itself lives
  account-wide, not per-app: a tri-state Language row in **Account Settings**
  (English / Dansk / Organization default; the third clears the preference so
  the org default takes over again). It survives re-login and follows the
  account across devices, and it never changes org or app defaults — it's a
  wish that's served where the current app supports it, falling back to the
  org/app default otherwise.

> **This is a bundle-mode move.** Running `apps.declare` organically against a
> live org with more than one `supported_locale` is **refused at declare time**
> (`multi_locale_requires_bundle`) — multi-locale apps are authored as bundles.
> See "Going bilingual is a bundle move" below.

## Move 2 — write display text as `@t:` message keys

In a **multilingual** App, every **declared display string** must be a message
key instead of literal text: the value starts with the sigil `@t:` and the rest
is a key you invent (`@t:customer.field.name`). The key is stored **verbatim**
— it is resolved against the viewer's locale at render, never baked to one
language at declare time. A monolingual App keeps its existing literal behavior.

Where the keys go — every declaration site from the earlier chapters accepts
one, because they are all opaque display text:

| Declaration site | Field(s) | Chapter |
|---|---|---|
| Property display label | `properties.<p>.label` | [objecttype-refinement.md](objecttype-refinement.md) |
| Enum value labels | `properties.<p>.valueLabels` | [objecttype-refinement.md](objecttype-refinement.md) |
| Type display name | `display.entityKind` / `display.entityKindPlural` | [objecttype-refinement.md](objecttype-refinement.md) |
| State labels | `states.<s>.label` on the StateMachine | [state-machines-binding.md](state-machines-binding.md) |
| Action presentation | `presentation.label` / `.description` / `.group` / `.confirm` / `.success` / `.pending` | base `canonify` skill, *declaring-actions* |
| View text | `title`, a column's `{ ref, label }`, a metric/button `label`, queue `title`/`why`/`empty`, filter-preset labels, … | [viewspec-quick-start.md](viewspec-quick-start.md) |
| Form field labels | form declarations | base `canonify` skill |
| Email templates | `subject` / `body` via `email_templates.declare` | below |

**Email templates.** `email_templates.declare` (`{ template_id, locale,
subject, body, expectedHash?, force? }`) registers a per-locale override for a
transactional email the platform sends — `template_id` names the well-known id
the send site references (e.g. `signature_request_invitation`), `locale` is
the closed `en`/`da` set, and a declared `(template_id, locale)` pair wins over
the built-in default for a recipient whose resolved locale matches. `subject`
and `body` are `{{param}}`-rendered at send time and stored byte-verbatim, so
either may be a `@t:` key (resolved at the same render seam against
`canon/i18n/<locale>.json`) or literal per-locale text written directly —
both are valid, and the gate below walks either form for completeness.
UPSERT keyed on `(template_id, locale)`; ships in the bundle under
`email-templates/` and round-trips through `catalog.export`/`apply`. A
re-declare of an EXISTING `(template_id, locale)` pair is guarded by
optimistic concurrency — a bare re-declare (no `expectedHash`/`force`) is
refused before any write (`CONFLICT/HASH_REQUIRED`); a FRESH pair has nothing
to conflict with. Same contract as `actions.update`'s `expectedHash`/`force`
(base skill's *error codes* reference).

Example — the enum from Chapter 2, bilingual:

```yaml
action: catalog.refine_object_type
input:
  name: customer
  display:
    entityKind: "@t:customer.type"
    entityKindPlural: "@t:customer.type_plural"
  properties:
    status:
      kind: mapped
      column: status
      type: enum
      values: [prospect, active, churned]        # NEVER localized — the API contract
      label: "@t:customer.field.status"
      valueLabels:
        prospect: "@t:customer.status.prospect"
        active: "@t:customer.status.active"
        churned: "@t:customer.status.churned"
```

**Why declared labels stop being optional in Danish.** With no declared label
the renderer *derives* one by humanizing the English identifier
(`customer_name` → "Customer Name", `customer` → "Customers"). That fallback
is structurally untranslatable — the source text *is* the English identifier,
and the auto-plural follows English rules (`kunde` would come out "kundes",
not "kunder"). So in a bilingual app, declare a label (or key) for **every**
rendered property, enum value, state, and type name. There is no shortcut.

**The explicit non-translating escape.** A brand name, an acronym such as
`CVR`, or another value that intentionally stays identical in every language
still has to be explicit: write an `@t:` key at the display site and put that
key in **every** declared locale bundle with the same value. For example,
`@t:company.cvr` may resolve to `CVR` in both `en.json` and `da.json`. This is
intentional — it records "this does not need translation" instead of allowing
a literal to escape translation accidentally. The parser's doubled sigil
(`@@t:foo`) remains a way to represent a literal that starts with `@t:`, but it
does **not** satisfy the multilingual display-literal advisory; use a
same-value message key at a user-facing display site.

**Key naming is yours.** Keys are a flat namespace you own; dots are just
convention. Pick a scheme (`<type>.<field>`, `<type>.status.<value>`,
`<type>.view.<name>`) and keep it consistent — the apply gate compares keys
byte-for-byte. The namespace is **canon-wide** (one message map per locale for
the whole canon, shared by every app in it), so prefix keys by domain or app
if the canon hosts more than one.

## Move 3 — ship `i18n/<locale>.json` in the bundle

Message files are a **bundle section**, next to `object-types/` and `views/`:
one file per locale at `i18n/<locale>.json`, holding the flat `key → text`
map plus the standard provenance fields:

```json
{
  "locale": "da",
  "managed_by": "tenant",
  "source_ref": null,
  "messages": {
    "customer.type": "Kunde",
    "customer.type_plural": "Kunder",
    "customer.field.name": "Navn",
    "customer.field.status": "Status",
    "customer.status.prospect": "Kundeemne",
    "customer.status.active": "Aktiv",
    "customer.status.churned": "Ophørt",
    "customer.view.list": "Kunder"
  }
}
```

Ship one for **every** declared locale — including your default. English is
not special: an `en.json` carries the English text under the same keys, so the
declarations themselves stay language-neutral.

There is **no organic verb for message bundles** — `i18n/*.json` exists only
in the manifest. You author it by editing the bundle directory (it is plain
declarative JSON, like everything else in the bundle) and `catalog.apply`-ing.
Values round-trip verbatim through export/apply — non-ASCII Danish, `{`
braces, even an escaped `@@t:` literal.

## The apply gate — what fails, and what the errors mean

When any App in the bundle declares `supported_locales`, `catalog.apply` (and
`canon catalog validate`, which reports **all** violations in one pass) runs
the three completeness checks below. When no App declares locales, none of the
completeness checks runs. In addition, a multilingual App (two or more declared
locales) gets the advisory `elegance_untranslatable_display_literal` for every
reachable fixed display string that is not an `@t:` key. Use
`canon catalog validate --strict` to make that advisory blocking for the
validation command; `catalog.apply` strips `elegance_*` findings from its apply
gate, so elegance never blocks an apply.

| Error `code` | Means | Fix |
|---|---|---|
| `default_locale_not_supported` | An App's `default_locale` is not in its own `supported_locales`. | Add it to the array, or change the default. |
| `missing_locale_bundle` | A declared locale ships no `i18n/<locale>.json` file. | Author the file — one per declared locale, including the default. |
| `missing_message_key` | A `@t:` key referenced by a declared artifact is absent from a declared locale's file. The error names **both the key and the locale**. | Add the key to that locale's `messages` map. |

The `missing_message_key` walk covers every `@t:` key in your declared
**object-types, actions, views, forms, state machines, email templates, and
app nav `section` headings**. A keyed section heading is gated exactly like a
view title: a missing translation fails the apply, naming the key and the
locale. The App's `name` is the one display string that renders **verbatim**
(never message-resolved), so don't put a `@t:` key there.

The literal advisory covers the fixed display sites in those reachable
artifacts: `display.entityKind`/`entityKindPlural`, mapped-property `label` and
`valueLabels`, ActionType presentation copy (including input labels and input
value labels), rollup labels, ViewSpec titles/labels/column labels/composite
filter labels, and App nav labels. It is scoped to the incoming bundle's owned
artifacts, so platform-owned catalog entries used as references do not become
the customer's findings.

Because the gate runs at apply, a translation gap is caught in CI (`canon
catalog validate ./my-app-canon` exits non-zero, naming every missing key),
not discovered by a Danish user staring at an English fallback. Once a bundle
applies green, a live render never misses.

## What each surface renders

Locale is resolved **per viewer, per surface** — each surface walks its own
preference chain and renders every declared string through the resolved
locale's messages:

| Surface | Locale chain |
|---|---|
| Web app (members) | user preference (Account Settings) → app/org default → platform default (`en`) |
| Transactional email | recipient's locale → default |
| Signing portal (external signers) | signer's locale → browser `Accept-Language` → envelope locale → default |
| Generated PDFs (contracts, certificates) | envelope locale → default |

So a Danish member sees Danish property labels, status chips, state labels,
type names, view titles, and action buttons/confirmations; a Danish signer
gets a Danish invitation email, portal, and completed-contract PDF — all from
the same bundle. Dates and numbers follow the viewer's locale convention
automatically (`48.000` for a Danish viewer, `48,000` for an English one);
declared property `currency` is unchanged — the *unit* travels with the data,
the *rendering convention* with the reader.

**What you do NOT translate: the platform chrome.** The shell around your app
— sign-in screens, "Save"/"Cancel", table paging, toasts, settings — is the
platform's own catalog, already localized in every platform locale and not
org-authorable. Your `i18n/*.json` covers exactly what your canon declares;
the two flip together when the user switches language.

## Going bilingual is a bundle move — organic multi-locale declares are refused

`apps.declare` invoked **organically** against a live org **refuses** more
than one `supported_locale`, with a `VALIDATION` error
(`multi_locale_requires_bundle`): organic authoring has no way to supply the
per-locale message files a multilingual app requires — `i18n/<locale>.json`
exists only in the manifest bundle. (This used to be a silent trap: the
organic declare succeeded and the failure surfaced later, at a
`catalog export` → `catalog apply` round-trip, as `missing_locale_bundle`.
It is now a hard error at declare time, where you can act on it.)
Single-locale and absent `supported_locales` are unaffected — both are
monolingual and owe no bundle.

The correct order — go bilingual **in bundle mode**:

1. `canon catalog export ./my-app-canon` — crystallize the current canon.
2. In the bundle, key your display strings (`@t:` in object-types, views,
   actions, state machines), author `i18n/en.json` + `i18n/da.json`, and set
   `supported_locales`/`default_locale` on the app — all in one edit.
3. `canon catalog validate ./my-app-canon` — the gate reports every missing
   key in one pass; iterate until clean.
4. `canon catalog apply ./my-app-canon --baseline <token>` — one
   atomic step: the app never exists in a locales-declared-but-untranslated
   state.

There is no organic route around this: even with message bundles already
applied, a multi-locale `supported_locales` on the App itself only lands via
`catalog.apply` — the app declaration rides the same bundle as its
translations, so the two can never drift apart.

## Worked example — the whole bundle at a glance

A complete minimal bilingual bundle is the fragments above assembled:

```
my-app-canon/
  manifest.json
  schema/customer.json          CREATE TABLE customer (…, status TEXT …)
  object-types/customer.json    @t: labels + valueLabels + entityKind/Plural
  actions/customer.activate.json  presentation.label/confirm/success as @t: keys
  views/Customer.list.json      "title": "@t:customer.view.list"
  views/Customer.detail.json    "title": "@t:customer.view.detail"
  apps/crm.json                 supported_locales ["en","da"], default_locale "en"
  i18n/en.json                  every key, in English
  i18n/da.json                  every key, in Danish
```

The two `i18n` files carry the **same key set** (the gate enforces it); only
the `messages` values differ. An action's localized presentation looks like:

```json
"presentation": {
  "label": "@t:customer.activate.label",
  "confirm": "@t:customer.activate.confirm",
  "success": "@t:customer.activate.success"
}
```

with `en.json` carrying `"customer.activate.label": "Activate customer"` and
`da.json` carrying `"customer.activate.label": "Aktivér kunde"`. The fastest
start: assemble exactly this shape, then run `canon catalog validate` until
green — it lists every key you still owe a locale.

## Recap

- `supported_locales` + `default_locale` on the App; ≥2 ⇒ renders in the
  viewer's Account Settings language preference when supported.
- Every display string can be `@t:<key>`; identifiers and enum values never.
- `i18n/<locale>.json` per declared locale, in the bundle, complete.
- `catalog.apply`/`validate` fail on `missing_message_key` (names key +
  locale), `missing_locale_bundle`, `default_locale_not_supported`.
- Go bilingual in bundle mode — an organic `apps.declare` of >1 locale is
  refused at declare time (`multi_locale_requires_bundle`).
