# Scenario patterns

Six worked, **harness-valid** scenarios. Each fenced `yaml` block below is a
complete `session.yaml` — it carries `name`, `principals`, and `steps`, and
decodes cleanly against the runner's `SessionSchema`.
The platform's CI parses and validates
every block on this page, and runs one of them end to end through the real
harness — so a copy-paste starts from a known-green state.

Copy a block, rename it, and edit the steps for your slice. The canonical
full-length reference is the worked `minimal-crm` scenario in the
Canonify platform repo.

---

## 1. CRM slice — table → ObjectType → drive the loop → navigate

The smallest end-to-end CRM proof: declare a table (auto-CRUD synthesises the
five CRUD verbs), refine the ObjectType to narrow `status` to an enum and add a
self-link, create two leads, and navigate. Mirrors the opening of the canonical
`minimal-crm` scenario.

```yaml
name: crm-slice
description: >
  Stand up a lead table, refine its ObjectType, create two leads through
  auto-CRUD, and read them back. Proves the table → ObjectType → business-loop
  path end to end through the public surface.
principals:
  agent:
    kind: agent
    id: agt_alice
    org_id: org_test
steps:
  - action: schema.propose_migration
    input:
      description: lead table — pre-conversion sales prospect
      sql: |
        CREATE TABLE lead (
          id          TEXT PRIMARY KEY,
          name        TEXT NOT NULL,
          email       TEXT NOT NULL,
          source      TEXT NOT NULL,
          status      TEXT NOT NULL DEFAULT 'new',
          created_at  TEXT NOT NULL,
          updated_at  TEXT NOT NULL
        )

  - action: catalog.refine_object_type
    input:
      name: lead
      description: Pre-conversion sales prospect.
      properties:
        status:
          kind: mapped
          column: status
          type: enum
          values: [new, contacted, qualified, converted, disqualified]

  - action: lead.create
    input:
      id: ld_REFERRAL01
      name: Acme Industries
      email: buyer@acme.example.com
      source: referral
      status: new
      created_at: '2026-05-11T12:00:00.000Z'
      updated_at: '2026-05-11T12:00:00.000Z'
    bind: referral
    expect:
      data:
        id: ld_REFERRAL01
        source: referral
        status: new

  - action: lead.create
    input:
      id: ld_WEBSITE01
      name: Globex Corp
      email: hello@globex.example.com
      source: website
      status: new
      created_at: '2026-05-11T12:00:00.000Z'
      updated_at: '2026-05-11T12:00:00.000Z'

  - read: objects.list
    input:
      type: lead
    expect:
      data: { length: 2 }

  - read: objects.get
    input:
      type: lead
      id: $referral.id
    expect:
      data:
        id: ld_REFERRAL01
        status: new
```

---

## 2. Invoice FSM lifecycle — register a state machine, walk it, prove the guard

Declare an invoice table, refine `status` to an enum, declare the transition
Actions, register the `Invoice.lifecycle` state machine, then walk
draft → sent → paid. Note the declaration order: the transition Actions land
**before** the SM that names them (the cross-validator resolves both ways).

```yaml
name: invoice-lifecycle
description: >
  Register the Invoice.lifecycle FSM and walk a draft invoice through sent to
  paid. The runtime auto-injects each state-column flip; the handlers only
  stamp domain timestamps.
principals:
  agent:
    kind: agent
    id: agt_alice
    org_id: org_test
steps:
  - action: schema.propose_migration
    input:
      description: invoice table — agent-issued bill awaiting payment
      sql: |
        CREATE TABLE invoice (
          id              TEXT PRIMARY KEY,
          customer_email  TEXT NOT NULL,
          amount_cents    INTEGER NOT NULL,
          status          TEXT NOT NULL DEFAULT 'draft',
          sent_at         TEXT,
          paid_at         TEXT,
          created_at      TEXT NOT NULL,
          updated_at      TEXT NOT NULL
        )

  - action: catalog.refine_object_type
    input:
      name: invoice
      properties:
        status:
          kind: mapped
          column: status
          type: enum
          values: [draft, sent, paid]

  - action: actions.declare
    input:
      name: invoice.send
      description: Send a draft invoice — Invoice.lifecycle → sent. Stamps sent_at.
      object_type: invoice
      input:
        invoice_id: { type: string }
      governance:
        requires_approval: false
        allowed_principals: [user, agent, service_account]
      idempotency: none
      transition:
        state_machine: Invoice.lifecycle
        from: [draft]
        to: sent
      handler:
        steps:
          - { op: update, object: invoice, where: { id: $input.invoice_id }, set: { sent_at: $now, updated_at: $now } }
        returns:
          invoice_id: $input.invoice_id

  - action: actions.declare
    input:
      name: invoice.mark_paid
      description: Mark an invoice paid — Invoice.lifecycle → paid (terminal).
      object_type: invoice
      input:
        invoice_id: { type: string }
      governance:
        requires_approval: false
        allowed_principals: [user, agent, service_account]
      idempotency: none
      transition:
        state_machine: Invoice.lifecycle
        from: [sent]
        to: paid
      handler:
        steps:
          - { op: update, object: invoice, where: { id: $input.invoice_id }, set: { paid_at: $now, updated_at: $now } }
        returns:
          invoice_id: $input.invoice_id

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
        sent:  { label: Sent }
        paid:  { label: Paid, terminal: true }
      transitions:
        - { via: invoice.send,      from: [draft], to: sent }
        - { via: invoice.mark_paid, from: [sent],  to: paid }

  - action: invoice.create
    input:
      id: inv_0001
      customer_email: buyer@acme.example.com
      amount_cents: 120000
      status: draft
      created_at: '2026-05-11T12:00:00.000Z'
      updated_at: '2026-05-11T12:00:00.000Z'

  - action: invoice.send
    input: { invoice_id: inv_0001 }

  # Illegal transition: mark_paid is only legal from `sent`. We are now `sent`,
  # so this is legal — walk it to `paid`.
  - action: invoice.mark_paid
    input: { invoice_id: inv_0001 }

  - read: objects.get
    input: { type: invoice, id: inv_0001 }
    expect:
      data:
        id: inv_0001
        status: paid
```

---

## 3. Multi-table atomic action — the `lead.convert` centerpiece

One declared handler that reads a lead, inserts a customer **and** a contact,
and updates the lead — all inside the runtime's single transaction. The FSM
auto-flips `lead.status` to `converted` in the same tx. This is the load-bearing
proof that the declarative handler DSL can express atomic multi-table writes
without TypeScript.

```yaml
name: lead-convert-atomic
description: >
  Declare lead.convert — an atomic three-table write (read lead → insert
  customer → insert contact → update lead) that runs in one tx alongside the
  FSM state flip. The expect block pins the returned ids by prefix.
principals:
  agent:
    kind: agent
    id: agt_alice
    org_id: org_test
steps:
  - action: schema.propose_migration
    input:
      description: lead table
      sql: |
        CREATE TABLE lead (
          id           TEXT PRIMARY KEY,
          name         TEXT NOT NULL,
          email        TEXT NOT NULL,
          status       TEXT NOT NULL DEFAULT 'qualified',
          customer_id  TEXT,
          converted_at TEXT,
          created_at   TEXT NOT NULL,
          updated_at   TEXT NOT NULL
        )

  - action: schema.propose_migration
    input:
      description: contact table
      sql: |
        CREATE TABLE contact (
          id          TEXT PRIMARY KEY,
          customer_id TEXT NOT NULL,
          name        TEXT NOT NULL,
          email       TEXT NOT NULL,
          role        TEXT,
          created_at  TEXT NOT NULL
        )

  # Expose the slice-1 customer's physical org_id column so the handler can
  # write it (typed-write validates against ObjectType properties).
  - action: catalog.refine_object_type
    input:
      name: customer
      description: Customer the org sells to.
      properties:
        org_id: { kind: mapped, column: org_id, type: string }

  - action: catalog.refine_object_type
    input:
      name: lead
      properties:
        status:
          kind: mapped
          column: status
          type: enum
          values: [qualified, converted]

  - action: actions.declare
    input:
      name: lead.convert
      description: Convert a qualified lead into a Customer + primary Contact.
      object_type: lead
      input:
        lead_id: { type: string }
      governance:
        requires_approval: false
        allowed_principals: [user, agent, service_account]
      idempotency: none
      transition:
        state_machine: Lead.status
        from: [qualified]
        to: converted
      handler:
        steps:
          - op: read
            object: lead
            where: { id: $input.lead_id }
            as: lead
          - op: insert
            object: customer
            values:
              name: $lead.name
              email: $lead.email
              org_id: $principal.org_id
            as: customer
          - op: insert
            object: contact
            values:
              customer_id: $customer.id
              name: $lead.name
              email: $lead.email
              role: primary
            as: contact
          - op: update
            object: lead
            where: { id: $input.lead_id }
            set:
              customer_id: $customer.id
              converted_at: $now
        returns:
          lead_id: $input.lead_id
          customer_id: $customer.id
          contact_id: $contact.id

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

  - action: lead.create
    input:
      id: ld_REFERRAL01
      name: Acme Industries
      email: buyer@acme.example.com
      status: qualified
      created_at: '2026-05-11T12:00:00.000Z'
      updated_at: '2026-05-11T12:00:00.000Z'

  - action: lead.convert
    input: { lead_id: ld_REFERRAL01 }
    bind: conversion
    expect:
      data:
        lead_id: ld_REFERRAL01
        customer_id: { matches: '^cus' }
        contact_id:  { matches: '^rec' }

  - read: objects.get
    input: { type: lead, id: ld_REFERRAL01 }
    expect:
      data:
        id: ld_REFERRAL01
        status: converted
```

---

## 4. Outbox-enqueue assertion — prove a hook enqueued, then delivered

An `on_enter` hook enqueues a welcome email when a lead converts. A
`check: outbox` step proves the `_outbox` row exists with the right payload; a
`tick: outbox` step then drains it, and a follow-up `check` proves it actually
*delivered* to the recording sink — not merely enqueued.

```yaml
name: outbox-enqueue
description: >
  A Lead.status on_enter[converted] hook enqueues an email.send_template row.
  Assert the enqueued payload, drain the outbox with a tick, and prove the
  effect was delivered to the recording sink.
principals:
  agent:
    kind: agent
    id: agt_alice
    org_id: org_test
steps:
  - action: schema.propose_migration
    input:
      description: lead table
      sql: |
        CREATE TABLE lead (
          id           TEXT PRIMARY KEY,
          name         TEXT NOT NULL,
          email        TEXT NOT NULL,
          status       TEXT NOT NULL DEFAULT 'qualified',
          customer_id  TEXT,
          created_at   TEXT NOT NULL,
          updated_at   TEXT NOT NULL
        )

  - action: catalog.refine_object_type
    input:
      name: lead
      properties:
        status:
          kind: mapped
          column: status
          type: enum
          values: [qualified, converted]

  - action: actions.declare
    input:
      name: lead.convert
      description: Convert a qualified lead — enqueues a welcome email on enter.
      object_type: lead
      input:
        lead_id: { type: string }
      governance:
        requires_approval: false
        allowed_principals: [user, agent, service_account]
      idempotency: none
      transition:
        state_machine: Lead.status
        from: [qualified]
        to: converted
      handler:
        steps:
          - { op: update, object: lead, where: { id: $input.lead_id }, set: { customer_id: $input.lead_id, updated_at: $now } }
        returns:
          lead_id: $input.lead_id

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
          # email.send_template validates { recipient, template_id, params? }.
          # The `inputs` envelope is unwrapped by the outbox drain at
          # re-invoke time; `$row.*` refs resolve against the post-transition
          # row at execute time (passthrough).
          - action: email.send_template
            inputs:
              recipient:   $row.email
              template_id: welcome

  - action: lead.create
    input:
      id: ld_REFERRAL01
      name: Acme Industries
      email: buyer@acme.example.com
      status: qualified
      created_at: '2026-05-11T12:00:00.000Z'
      updated_at: '2026-05-11T12:00:00.000Z'

  - action: lead.convert
    input: { lead_id: ld_REFERRAL01 }

  # The row is enqueued but not yet delivered. The stored payload is the
  # FSM hook's wrapped `{ inputs: { … } }` envelope, so the matcher descends
  # through `inputs`.
  - check: outbox
    where: { action: email.send_template, kind: effect }
    expect:
      count: 1
      payload:
        inputs:
          template_id: welcome
          recipient: buyer@acme.example.com

  # Drain the outbox — re-invokes the enqueued effect into the recording sink.
  # The tick's `expect` matches the DrainResult under `data`.
  - tick: outbox
    expect:
      data:
        delivered: 1
        deadlettered: 0

  # Now the row is done AND the email reached the sink.
  - check: outbox
    where: { action: email.send_template, kind: effect }
    expect:
      status: done
      delivered:
        email: { length: 1 }
```

---

## 5. Governance / FSM-guard denial — assert the failure, prove no mutation

Two denial flavours. First, an FSM guard rejects an illegal transition with
`INVALID_STATE`. Second, a governance handler asserts a visibility precondition
and raises `FORBIDDEN`. In both cases the `expect: { error: { code: … } }`
block flips the step into denial mode (the action MUST fail), and a follow-up
read proves the denied call did not mutate the row.

```yaml
name: governance-denial
description: >
  Prove the runtime denies illegal calls. The FSM guard rejects an out-of-state
  transition (INVALID_STATE); a handler-level visibility assert rejects a
  non-owner (FORBIDDEN). Both leave the row unchanged.
principals:
  owner:
    kind: service_account
    id: sa_owner
    org_id: org_test
  intruder:
    kind: service_account
    id: sa_intruder
    org_id: org_test
steps:
  # This session has two principals, so EVERY step names one explicitly —
  # the runner only defaults `principal:` when there is a single principal
  # (or one named `agent`).
  - action: schema.propose_migration
    principal: owner
    input:
      description: ticket table
      sql: |
        CREATE TABLE ticket (
          id                    TEXT PRIMARY KEY,
          owner_principal_id    TEXT NOT NULL,
          title                 TEXT NOT NULL,
          status                TEXT NOT NULL DEFAULT 'open',
          created_at            TEXT NOT NULL,
          updated_at            TEXT NOT NULL
        )

  - action: catalog.refine_object_type
    principal: owner
    input:
      name: ticket
      properties:
        status:
          kind: mapped
          column: status
          type: enum
          values: [open, in_progress, done]

  - action: actions.declare
    principal: owner
    input:
      name: ticket.start
      description: Move an open ticket to in_progress — owner only.
      object_type: ticket
      input:
        ticket_id: { type: string }
      governance:
        requires_approval: false
        allowed_principals: [service_account]
      idempotency: none
      transition:
        state_machine: Ticket.status
        from: [open]
        to: in_progress
      handler:
        steps:
          # Visibility guard — only the ticket's owner may start it.
          - op: read_optional
            object: ticket
            where: { id: $input.ticket_id, owner_principal_id: $principal.id }
            as: ticket
          - op: assert
            when: { ref: $ticket.id, present: true }
            else:
              code: FORBIDDEN
              message: 'You do not own this ticket'
          - { op: update, object: ticket, where: { id: $input.ticket_id }, set: { updated_at: $now } }
        returns:
          ticket_id: $input.ticket_id

  - action: actions.declare
    principal: owner
    input:
      name: ticket.complete
      description: Move an in_progress ticket to done.
      object_type: ticket
      input:
        ticket_id: { type: string }
      governance:
        requires_approval: false
        allowed_principals: [service_account]
      idempotency: none
      transition:
        state_machine: Ticket.status
        from: [in_progress]
        to: done
      handler:
        steps:
          - { op: update, object: ticket, where: { id: $input.ticket_id }, set: { updated_at: $now } }
        returns:
          ticket_id: $input.ticket_id

  - action: state_machines.register
    principal: owner
    input:
      kind: state_machine
      track: customer
      name: Ticket.status
      objectType: ticket
      property: status
      initial: open
      states:
        open:        { label: Open }
        in_progress: { label: In progress }
        done:        { label: Done, terminal: true }
      transitions:
        - { via: ticket.start,    from: [open],        to: in_progress }
        - { via: ticket.complete, from: [in_progress], to: done }

  - action: ticket.create
    input:
      id: tk_0001
      owner_principal_id: sa_owner
      title: Fix the thing
      status: open
      created_at: '2026-05-11T12:00:00.000Z'
      updated_at: '2026-05-11T12:00:00.000Z'
    principal: owner

  # DENIAL 1 — FSM guard. complete is only legal from in_progress; the ticket
  # is open, so the auto-injected guard rejects with INVALID_STATE.
  - action: ticket.complete
    input: { ticket_id: tk_0001 }
    principal: owner
    expect:
      error:
        code: INVALID_STATE

  # DENIAL 2 — governance. A non-owner tries to start the ticket; the handler's
  # visibility assert raises FORBIDDEN.
  - action: ticket.start
    input: { ticket_id: tk_0001 }
    principal: intruder
    expect:
      error:
        code: FORBIDDEN

  # Neither denied call mutated the row — it is still open.
  - read: objects.get
    principal: owner
    input: { type: ticket, id: tk_0001 }
    expect:
      data:
        id: tk_0001
        status: open

  # The audit log records the denials (status='denied' rows survive rollback).
  - check: audit
    where: { action: ticket.start, principal: sa_intruder }
    expect:
      count: 1
```

---

## 6. Aggregate reads — grouped, time-bucketed, and multi-series (`objects.aggregate`)

Refine `created_at` to `timestamp` (auto-CRUD infers it as a plain `string`,
and a string cannot be time-bucketed), create a few purchases across two
months and two statuses, then prove `objects.aggregate`'s `group_by_bucket`
and `series_by` params — the exact read a `metric`/`chart`/grouped-`table`
ViewSpec binds to (see the `canonify-viewspecs` skill). `count`/`sum`
zero-fill **interior** gaps in the bucketed spine (a bucket with no rows is a
real `0`); with `series_by` each series walks its **own** spine, so the
single `paid` purchase does not drag a phantom `0` into February or March.

A bucketed `objects.aggregate` read also stamps `partial: true` on the group
whose bucket contains the read's current instant — the in-progress trailing
period (tick xz2; full detail in the `canonify-viewspecs` skill's
`viewspec-analytics.md`, *Comparison layers*). It ships on **every**
bucketed read, whether or not the group it lands on is even shown below, so
the `groups:` expectation below names only `group` and `value` — a plain
(non-`equals`) array expectation is SUBSET-tolerant per element (see
`scenario-structure.md`, *The `expect:` matcher grammar*), so it keeps
passing regardless of which bucket "now" falls in.

```yaml
name: aggregate-reads
description: >
  Prove objects.aggregate's grouped, time-bucketed (group_by_bucket), and
  multi-series (series_by) read end to end — the data primitive
  metric/chart/grouped-table ViewSpecs bind to.
principals:
  agent:
    kind: agent
    id: agt_alice
    org_id: org_test
steps:
  - action: schema.propose_migration
    input:
      description: purchase table — a customer order
      sql: |
        CREATE TABLE purchase (
          id              TEXT PRIMARY KEY,
          customer_email  TEXT NOT NULL,
          amount_cents    INTEGER NOT NULL,
          status          TEXT NOT NULL DEFAULT 'draft',
          created_at      TEXT NOT NULL,
          updated_at      TEXT NOT NULL
        )

  - action: catalog.refine_object_type
    input:
      name: purchase
      properties:
        status:
          kind: mapped
          column: status
          type: enum
          values: [draft, paid]
        # Auto-CRUD infers created_at as a plain `string` (the column is
        # TEXT); time-bucketing requires the honest `timestamp` type, exactly
        # as view_specs.declare requires for a chart/metric trend.
        created_at:
          kind: mapped
          column: created_at
          type: timestamp

  - action: purchase.create
    input:
      id:             pur_0001
      customer_email: ap@acme.example.com
      amount_cents:   10000
      status:         draft
      created_at:     '2026-01-10T09:00:00.000Z'
      updated_at:     '2026-01-10T09:00:00.000Z'

  - action: purchase.create
    input:
      id:             pur_0002
      customer_email: ap@acme.example.com
      amount_cents:   5000
      status:         paid
      created_at:     '2026-01-20T09:00:00.000Z'
      updated_at:     '2026-01-20T09:00:00.000Z'

  - action: purchase.create
    input:
      id:             pur_0003
      customer_email: ap@globex.example.com
      amount_cents:   20000
      status:         draft
      created_at:     '2026-03-05T09:00:00.000Z'
      updated_at:     '2026-03-05T09:00:00.000Z'

  # Revenue per month. The keys are bucket strings ("2026-01"), not raw
  # timestamps, and 2026-02 is zero-filled — an interior gap between the
  # first and last OBSERVED month, not an absent row.
  - read: objects.aggregate
    input:
      type:            purchase
      op:              sum
      property:        amount_cents
      group_by:        created_at
      group_by_bucket: month
    expect:
      data:
        groups:
          - { group: { created_at: '2026-01' }, value: 15000 }
          - { group: { created_at: '2026-02' }, value: 0 }
          - { group: { created_at: '2026-03' }, value: 20000 }

  # The same read, split by status — the shape a multi-series chart needs.
  # `draft` walks Jan through Mar (so Feb zero-fills); `paid` has exactly one
  # observation in Jan and does NOT inherit `draft`'s Feb/Mar range — proof
  # that zero-fill is per-series, not a shared spine.
  - read: objects.aggregate
    input:
      type:            purchase
      op:              sum
      property:        amount_cents
      group_by:        created_at
      group_by_bucket: month
      series_by:       status
    expect:
      data:
        groups:
          - { group: { created_at: '2026-01', status: draft }, value: 10000 }
          - { group: { created_at: '2026-02', status: draft }, value: 0 }
          - { group: { created_at: '2026-03', status: draft }, value: 20000 }
          - { group: { created_at: '2026-01', status: paid }, value: 5000 }
```
