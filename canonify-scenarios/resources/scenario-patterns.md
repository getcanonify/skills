# Scenario patterns

Five complete `session.yaml` documents that decode against the runner's
`SessionSchema` and run green; copy one, rename it, edit the steps.

## 1. CRM slice — table → ObjectType → create → read

```yaml
name: crm-slice
principals:
  owner: { kind: user, id: usr_owner, org_id: org_test }
steps:
  - action: schema.propose_migration
    input:
      description: lead table — pre-conversion sales prospect
      sql: |
        CREATE TABLE lead (
          id TEXT PRIMARY KEY, name TEXT NOT NULL, email TEXT NOT NULL,
          source TEXT NOT NULL, status TEXT NOT NULL DEFAULT 'new',
          created_at TEXT NOT NULL, updated_at TEXT NOT NULL)
  - action: catalog.refine_object_type
    input:
      name: lead
      properties:
        status: { kind: mapped, column: status, type: enum, values: [new, contacted, qualified, converted] }
  - action: lead.create
    input: { id: ld_CRM_REFERRAL, name: Acme Industries, email: buyer@acme.example.com, source: referral }
    bind: referral
    expect: { data: { id: ld_CRM_REFERRAL, status: new } }
  - read: objects.list
    input: { type: lead, filter: { source: referral } }
    expect: { data: { length: 1 } }
  - read: objects.get
    input: { type: lead, id: $referral.id }
    expect: { data: { source: referral, status: new } }
```

## 2. Invoice FSM lifecycle — declare transitions, register the machine, walk it

Actions before the machine that names them; handlers stamp times, the runtime writes `status`.

```yaml
name: invoice-lifecycle
principals:
  owner: { kind: user, id: usr_owner, org_id: org_test }
steps:
  - action: schema.propose_migration
    input:
      description: invoice table
      sql: |
        CREATE TABLE invoice (
          id TEXT PRIMARY KEY, customer_email TEXT NOT NULL, amount_cents INTEGER NOT NULL,
          status TEXT NOT NULL DEFAULT 'draft', sent_at TEXT, paid_at TEXT,
          created_at TEXT NOT NULL, updated_at TEXT NOT NULL)
  - action: catalog.refine_object_type
    input:
      name: invoice
      properties:
        status: { kind: mapped, column: status, type: enum, values: [draft, sent, paid] }
  - action: actions.declare
    input:
      name: invoice.send
      description: Send a draft invoice. Stamps sent_at.
      object_type: invoice
      input: { invoice_id: { type: string } }
      governance: { requires_approval: false, allowed_principals: [user, agent, service_account] }
      idempotency: none
      transition: { state_machine: Invoice.lifecycle, from: [draft], to: sent }
      handler:
        steps:
          - { op: update, object: invoice, where: { id: $input.invoice_id }, set: { sent_at: $now } }
        returns: { invoice_id: $input.invoice_id }
  - action: actions.declare
    input:
      name: invoice.mark_paid
      description: Mark a sent invoice paid.
      object_type: invoice
      input: { invoice_id: { type: string } }
      governance: { requires_approval: false, allowed_principals: [user, agent, service_account] }
      idempotency: none
      transition: { state_machine: Invoice.lifecycle, from: [sent], to: paid }
      handler:
        steps:
          - { op: update, object: invoice, where: { id: $input.invoice_id }, set: { paid_at: $now } }
        returns: { invoice_id: $input.invoice_id }
  - action: state_machines.register
    input:
      kind: state_machine
      track: customer
      name: Invoice.lifecycle
      objectType: invoice
      property: status
      initial: draft
      states:
        draft: { label: Draft }
        sent: { label: Sent }
        paid: { label: Paid, terminal: true }
      transitions:
        - { via: invoice.send, from: [draft], to: sent }
        - { via: invoice.mark_paid, from: [sent], to: paid }
  - action: invoice.create
    input: { id: inv_0001, customer_email: buyer@acme.example.com, amount_cents: 120000 }
    expect: { data: { status: draft } }
  - action: invoice.send
    input: { invoice_id: inv_0001 }
  - action: invoice.mark_paid
    input: { invoice_id: inv_0001 }
  - read: objects.get
    input: { type: invoice, id: inv_0001 }
    expect: { data: { id: inv_0001, status: paid } }
```

## 3. Atomic multi-table action + outbox hook — `lead.convert`

One transaction: read the lead, insert an account and a contact, flip `status`;
the `on_enter` hook enqueues an email. `check: outbox` proves it was enqueued
(the payload is the hook's `{ inputs: … }` wrapper), `tick: outbox` drains it,
`delivered:` proves it reached the sink.

```yaml
name: lead-convert-outbox
principals:
  owner: { kind: user, id: usr_owner, org_id: org_test }
steps:
  - action: schema.propose_migration     # one statement per migration
    input:
      description: lead table
      sql: |
        CREATE TABLE lead (
          id TEXT PRIMARY KEY, name TEXT NOT NULL, email TEXT NOT NULL,
          status TEXT NOT NULL DEFAULT 'qualified', account_id TEXT, converted_at TEXT,
          created_at TEXT NOT NULL, updated_at TEXT NOT NULL)
  - action: schema.propose_migration
    input:
      description: account table
      sql: |
        CREATE TABLE account (
          id TEXT PRIMARY KEY, name TEXT NOT NULL, email TEXT NOT NULL,
          created_at TEXT NOT NULL, updated_at TEXT NOT NULL)
  - action: schema.propose_migration
    input:
      description: contact table
      sql: |
        CREATE TABLE contact (
          id TEXT PRIMARY KEY, account_id TEXT NOT NULL, name TEXT NOT NULL,
          email TEXT NOT NULL, role TEXT, created_at TEXT NOT NULL, updated_at TEXT NOT NULL)
  - action: catalog.refine_object_type
    input:
      name: lead
      properties:
        status: { kind: mapped, column: status, type: enum, values: [qualified, converted] }
  - action: actions.declare
    input:
      name: lead.convert
      description: Convert a qualified lead into an account and a primary contact.
      object_type: lead
      input: { lead_id: { type: string } }
      governance: { requires_approval: false, allowed_principals: [user, agent, service_account] }
      idempotency: none
      transition: { state_machine: Lead.status, from: [qualified], to: converted }
      handler:
        steps:
          - { op: read, object: lead, where: { id: $input.lead_id }, as: lead }
          - { op: insert, object: account, values: { name: $lead.name, email: $lead.email }, as: account }
          - op: insert
            object: contact
            values: { account_id: $account.id, name: $lead.name, email: $lead.email, role: primary }
            as: contact
          - { op: update, object: lead, where: { id: $input.lead_id }, set: { account_id: $account.id, converted_at: $now } }
        returns: { lead_id: $input.lead_id, account_id: $account.id, contact_id: $contact.id }
  - action: state_machines.register
    input:
      kind: state_machine
      track: customer
      name: Lead.status
      objectType: lead
      property: status
      initial: qualified
      states:
        qualified: { label: Qualified }
        converted: { label: Converted, terminal: true }
      transitions:
        - { via: lead.convert, from: [qualified], to: converted }
      on_enter:
        converted:
          - action: email.send_template
            inputs: { recipient: $row.email, template_id: welcome }
  - action: lead.create
    input: { id: ld_CONV_ACME, name: Acme Industries, email: buyer@acme.example.com }
  - action: lead.convert
    input: { lead_id: ld_CONV_ACME }
    expect:
      data: { lead_id: ld_CONV_ACME, account_id: { matches: '^rec_' }, contact_id: { matches: '^rec_' } }
  - read: objects.get
    input: { type: lead, id: ld_CONV_ACME }
    expect: { data: { status: converted } }
  - check: outbox
    where: { action: email.send_template, kind: effect }
    expect:
      count: 1
      payload: { inputs: { template_id: welcome, recipient: buyer@acme.example.com } }
  - tick: outbox
    expect: { data: { delivered: 1, deadlettered: 0 } }
  - check: outbox
    where: { action: email.send_template, kind: effect }
    expect: { status: done, delivered: { email: { length: 1 } } }
```

## 4. Denials — governance and the FSM guard, with no mutation

`expect: { error: … }` flips a step into denial mode (the action MUST fail). A
multi-principal session names `principal:` on every step.

```yaml
name: governance-denial
principals:
  owner: { kind: service_account, id: sa_owner, org_id: org_test }
  intruder: { kind: service_account, id: sa_intruder, org_id: org_test }
steps:
  - action: schema.propose_migration
    principal: owner
    input:
      description: ticket table
      sql: |
        CREATE TABLE ticket (
          id TEXT PRIMARY KEY, owner_principal_id TEXT NOT NULL, title TEXT NOT NULL,
          status TEXT NOT NULL DEFAULT 'open', created_at TEXT NOT NULL, updated_at TEXT NOT NULL)
  - action: catalog.refine_object_type
    principal: owner
    input:
      name: ticket
      properties:
        status: { kind: mapped, column: status, type: enum, values: [open, in_progress] }
  - action: actions.declare
    principal: owner
    input:
      name: ticket.start
      description: Start an open ticket — owner only.
      object_type: ticket
      input: { ticket_id: { type: string } }
      governance: { requires_approval: false, allowed_principals: [service_account] }
      idempotency: none
      transition: { state_machine: Ticket.status, from: [open], to: in_progress }
      handler:
        steps:
          - { op: read_optional, object: ticket, where: { id: $input.ticket_id, owner_principal_id: $principal.id }, as: ticket }
          - { op: assert, when: { ref: $ticket.id, present: true }, else: { code: FORBIDDEN, message: You do not own this ticket } }
        returns: { ticket_id: $input.ticket_id }
  - action: state_machines.register
    principal: owner
    input:
      kind: state_machine
      track: customer
      name: Ticket.status
      objectType: ticket
      property: status
      initial: open
      states: { open: { label: Open }, in_progress: { label: In progress, terminal: true } }
      transitions:
        - { via: ticket.start, from: [open], to: in_progress }
  - action: ticket.create
    principal: owner
    input: { id: tk_DENY_0001, owner_principal_id: sa_owner, title: Fix the thing }
  - action: ticket.start             # handler assert: not the owner
    principal: intruder
    input: { ticket_id: tk_DENY_0001 }
    expect: { error: { code: FORBIDDEN } }
  - read: objects.get                # the denied call changed nothing
    principal: owner
    input: { type: ticket, id: tk_DENY_0001 }
    expect: { data: { status: open } }
  - check: audit                     # denied rows survive the rollback
    where: { action: ticket.start, principal: sa_intruder }
    expect: { count: 1 }
  - action: ticket.start
    principal: owner
    input: { ticket_id: tk_DENY_0001 }
  - action: ticket.start             # FSM guard: no longer `open`
    principal: owner
    input: { ticket_id: tk_DENY_0001 }
    expect: { error: { code: INVALID_STATE } }
```

## 5. Aggregate reads — bucketed and multi-series

The read `metric` / `chart` / grouped `table` views bind to. Interior gaps
zero-fill; each `series_by` series walks its own spine (no phantom February for
`paid`). The group holding "now" also carries `partial: true`, which a subset
match ignores.

```yaml
name: aggregate-reads
principals:
  owner: { kind: user, id: usr_owner, org_id: org_test }
steps:
  - action: schema.propose_migration
    input:
      description: purchase table
      sql: |
        CREATE TABLE purchase (
          id TEXT PRIMARY KEY, amount_cents INTEGER NOT NULL,
          status TEXT NOT NULL DEFAULT 'draft', created_at TEXT NOT NULL, updated_at TEXT NOT NULL)
  - action: catalog.refine_object_type
    input:
      name: purchase
      properties:
        status: { kind: mapped, column: status, type: enum, values: [draft, paid] }
  - action: purchase.create
    input: { id: pur_AGG_1, amount_cents: 10000, status: draft, created_at: '2026-01-10T09:00:00.000Z' }
  - action: purchase.create
    input: { id: pur_AGG_2, amount_cents: 5000, status: paid, created_at: '2026-01-20T09:00:00.000Z' }
  - action: purchase.create
    input: { id: pur_AGG_3, amount_cents: 20000, status: draft, created_at: '2026-03-05T09:00:00.000Z' }
  - read: objects.aggregate
    input: { type: purchase, op: sum, property: amount_cents, group_by: created_at, group_by_bucket: month }
    expect:
      data:
        groups:
          - { group: { created_at: '2026-01' }, value: 15000 }
          - { group: { created_at: '2026-02' }, value: 0 }
          - { group: { created_at: '2026-03' }, value: 20000 }
  - read: objects.aggregate
    input: { type: purchase, op: sum, property: amount_cents, group_by: created_at, group_by_bucket: month, series_by: status }
    expect:
      data:
        groups:
          - { group: { created_at: '2026-01', status: draft }, value: 10000 }
          - { group: { created_at: '2026-02', status: draft }, value: 0 }
          - { group: { created_at: '2026-03', status: draft }, value: 20000 }
          - { group: { created_at: '2026-01', status: paid }, value: 5000 }
```
