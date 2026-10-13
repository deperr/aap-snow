# Role: `snow_ticket`

Every ServiceNow write in this repo goes through this role. That is
[design decision 2](../../docs/mvp/architecture.md#2-never-touch-the-ticket-outside-the-snow_ticket-role):
when the ticket type changes (incident → `sc_req_item` in
[Phase 6](../../docs/mvp/phase-6-servicenow-trigger.md), or `change_request` in
[Phase 8](../../docs/mvp/phase-8-options.md)), only this directory changes.

The role has **no default action**. Always name one with `tasks_from:`, so that grepping
for `tasks_from:` finds every place a ticket is touched.

## Actions

### `open` — create or adopt the ticket

```yaml
- name: Open the ticket
  ansible.builtin.include_role:
    name: snow_ticket
    tasks_from: open
  vars:
    snow_ticket_short_description: "Provision & patch EC2 instance: web-demo-01"
    snow_ticket_description: "…"
    snow_ticket_opening_note: "Request received."   # optional
```

Sets `snow_ticket_sys_id` and `snow_ticket_number` as facts.

If `snow_sys_id` is supplied and non-empty, the role **adopts** that ticket instead of
creating one, looks up its number, and asserts it exists. Nothing supplies `snow_sys_id`
during the MVP — that is the
[Phase 6 hook](../../docs/mvp/architecture.md#1-make-node-1-skippable), wired now because
it costs nothing.

### `note` — append a work note

```yaml
- name: Say what we're doing
  ansible.builtin.include_role:
    name: snow_ticket
    tasks_from: note
  vars:
    snow_ticket_note: "Instance reachable. Applying all available updates…"
```

Takes the ticket from `snow_ticket_sys_id` (same play) or `snow_sys_id` (workflow
artifact), in that order.

### Not yet implemented

`resolve` and `escalate` land in
[Phase 4](../../docs/mvp/phase-4-workflow.md) and
[Phase 5](../../docs/mvp/phase-5-failure-and-teardown.md).

## Variables

| Variable | Default | Purpose |
|---|---|---|
| `snow_ticket_caller` | `admin` | Ticket caller. Override with a real PDI user. |
| `snow_ticket_impact` | `low` | Creation impact. |
| `snow_ticket_urgency` | `low` | Creation urgency. |
| `snow_ticket_delegate_host` | `localhost` | Where API calls execute. |
| `snow_sys_id` | `""` | Adopt this ticket instead of creating one. |

## Why every task is `delegate_to: localhost`

From [Phase 3](../../docs/mvp/phase-3-patch-instance.md): once plays start running against
the EC2 instance, an undelegated `servicenow.itsm` task executes *on that instance* — so
the API call egresses from AWS and needs the collection in the instance's Python
environment. It fails in a confusing way. Delegating inside the role means no caller can
forget.

`run_once: true` is paired with it so a play against N hosts still writes one note, not N.

`become: false` is set for the same reason. localhost here is the execution environment
container, which has no `sudo`, and a REST call needs no privilege. A caller that sets
`become: true` at play level — as a patching play naturally does — would otherwise fail
these tasks with `sudo: command not found`.

## Authentication

The modules read `SN_HOST`, `SN_USERNAME`, `SN_PASSWORD` from the environment. In AAP these
come from the ServiceNow credential ([Phase 0](../../docs/mvp/phase-0-environment.md));
locally, export them before running. No `instance:` block is needed anywhere.
