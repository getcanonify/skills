---
title: Go bilingual — locales, @t: message keys, and the i18n bundle
description: Make an app render in two languages — declare supported_locales/default_locale on the App, write display text as @t:<key> message keys, ship one i18n/<locale>.json message file per locale in the bundle, and let catalog.apply fail the build when a translation is missing.
---

# Go bilingual

Locale is presentation: it selects which declared display text renders, per
viewer, and never changes what the API returns. Identifiers — action, property
and view names, enum **values** — stay English `snake_case` in every locale; you
translate what a value reads as (`valueLabels`), never the value. Who owns which
words on a screen (your catalog vs. the platform chrome) is one table in
[`canonify-viewspecs`](../../canonify-viewspecs/SKILL.md).

| Move | Where | What |
|---|---|---|
| 1. Declare support | the App | `supported_locales: ["en", "da"]` + `default_locale: "en"` |
| 2. Key your text | every display string | `@t:<your.key>` instead of literal text |
| 3. Ship the words | the bundle: `i18n/<locale>.json` | one flat `key → text` map per declared locale |

Absent `supported_locales`, or one entry, the app is monolingual: literals
render as written, no message files. Declare two only if you will maintain both.

## 1 — `supported_locales` on the App

`supported_locales` is the closed set `en`, `da` (anything else fails decode).
`default_locale` must be named explicitly and be a member
(`default_locale_not_supported`, on `apps.declare` as in a bundle). With two
or more, the app renders in the viewer's language preference — an
account-wide row in Account Settings (English / Dansk / Organization default),
not a per-app switcher — falling back to the app/org default.

**This is a bundle move.** An organic `apps.declare` with more than one locale is
refused (`VALIDATION`, `multi_locale_requires_bundle`): message files exist only
in a bundle. Export, key the strings, author `i18n/en.json` + `i18n/da.json`, set
the locales — one edit — then `canon catalog validate` until clean and
`canon catalog apply <dir> --baseline <token>`.

## 2 — `@t:` keys

In a multilingual App every display string reachable from its nav must be an
`@t:` key (there is no separate literal-text advisory). The key is stored
verbatim and resolved per viewer at render. Sites that take one: property
`label` and `valueLabels`, `display.entityKind` / `entityKindPlural`, state
`label`s, action `presentation` copy (incl. `inputLabels` /
`inputValueLabels`), rollup labels, view titles / column labels / metric and
button labels / queue copy / filter labels, form fields, nav `section`
headings, and email templates. The App's `name` renders verbatim — never key it.

```yaml
action: catalog.refine_object_type
input:
  name: customer
  display: { entityKind: "@t:customer.type", entityKindPlural: "@t:customer.type_plural" }
  properties:
    status:
      kind: mapped
      column: status
      type: enum
      values: [prospect, active, churned]     # never localized
      label: "@t:customer.field.status"
      valueLabels: { prospect: "@t:customer.status.prospect", active: "@t:customer.status.active", churned: "@t:customer.status.churned" }
```

- **In Danish, declared labels are mandatory.** The undeclared fallback
  humanizes the English identifier (`customer_name` → "Customer Name", plural
  by English rules) and cannot be translated.
- **Text that stays the same in every language** (a brand, `CVR`) is still a key,
  with the same value in every locale file. `@@t:foo` renders the literal `@t:foo`.
- **Keys are a flat namespace you own, canon-wide** (one map per locale shared by
  every app) — prefix by domain or app.

**Email templates.** `email_templates.declare { template_id, locale, subject,
body, expectedHash?, force? }` overrides a platform email (e.g.
`signature_request_invitation`) for recipients of that locale. `subject` / `body`
are `{{param}}`-rendered at send, and may be `@t:` keys or literal per-locale
text. Upsert on `(template_id, locale)`; bundled under `email-templates/`. A
re-declare of an existing pair needs `expectedHash` or `force`
(`CONFLICT/HASH_REQUIRED`).

## 3 — `i18n/<locale>.json`

```json
{
  "locale": "da",
  "managed_by": "tenant",
  "source_ref": null,
  "messages": {
    "customer.type": "Kunde",
    "customer.type_plural": "Kunder",
    "customer.field.status": "Status",
    "customer.status.prospect": "Kundeemne",
    "customer.status.active": "Aktiv",
    "customer.status.churned": "Ophørt"
  }
}
```

One per declared locale, including the default (`en.json` carries the English
under the same keys). No organic verb writes these; values round-trip verbatim.

## The apply gate

When any App declares `supported_locales`, `catalog.apply` and
`canon catalog validate` (all violations, one pass) refuse:

| `code` | Fix |
|---|---|
| `default_locale_not_supported` | Add the default to `supported_locales`, or change it. |
| `missing_locale_bundle` | Author `i18n/<locale>.json` for that locale. |
| `missing_message_key` (names key and locale) | Add the key to that locale's `messages`. |

The key walk covers object-types, actions, views, forms, state machines, email
templates and nav section headings of the incoming bundle (platform-owned
entries are not your findings). A bundle that applies green never misses a key
at render.

## What each surface renders

| Surface | Locale chain |
|---|---|
| Web app (members) | user preference → app/org default → `en` |
| Transactional email | recipient's locale → default |
| Signing portal | signer's locale → browser `Accept-Language` → envelope locale → default |
| Generated PDFs | envelope locale → default |

Dates and numbers follow the viewer's convention (`48.000` vs `48,000`); a
property's `currency` travels with the data. The platform chrome around your app
(sign-in, Save/Cancel, paging, toasts) is the platform's own, already localized,
and switches with the user's language.
