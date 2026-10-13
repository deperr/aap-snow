# Phase 4 — Wire it into one AAP workflow

[← Phase 3](phase-3-patch-instance.md) · [Index](README.md) · [Phase 5 →](phase-5-failure-and-teardown.md)

**Effort:** ~1 day · **Depends on:** Phases [1](phase-1-open-ticket.md)–[3](phase-3-patch-instance.md)

## Goal

One launch, one ticket, one instance — the whole story in a single workflow job template.

## Node graph

| Node | Job template | Edge from parent |
|---|---|---|
| 1 | `snow-open-ticket` | — |
| 2 | `provision-ec2` | 1, on success |
| 3 | `patch-instance` | 2, on success |
| 4 | `reboot-and-verify` | 3, on success |
| 5 | `snow-close-ticket` | 4, on success |
| E | `snow-escalate` | 2, 3 **and** 4, on failure |

Add a survey on the workflow: `ec2_name`, `ec2_instance_type` (choice list:
`t3.micro`/`t3.small`), `ec2_env`. Node 4 reboots only `when: reboot_required | bool`, then
re-runs `wait_for_connection` and a trivial health check.

`snow-escalate` is built in [Phase 5](phase-5-failure-and-teardown.md); until then, leave
the failure edges off and the workflow simply stops on failure with the ticket open.

### Why node 5 is not wired "always"

An earlier draft of this plan had node 5 on an **always** edge from node 4, branching on a
`workflow_outcome` variable. That is broken twice over, and it is worth stating plainly
because the mistake is an easy one:

- An `always` edge from node 4 only fires **if node 4 ran at all**. When node 2 fails,
  nodes 3 and 4 never execute, so node 5 never executes either — and the ticket is left
  open with no closing note, which is precisely the case the failure path exists for.
- Nothing ever set `workflow_outcome`. The `when:` would have silently taken the success
  branch and resolved tickets for failed runs.

The model above avoids both: **every path terminates in exactly one ticket-writing node.**
Success ends at node 5 (resolved); failure ends at `snow-escalate` (open, escalated). No
outcome variable is needed, because the graph itself encodes the outcome.

## Node 5 — close the ticket

Reached only on success, so it needs no conditional:

```yaml
- name: Resolve the incident
  ansible.builtin.include_role:
    name: snow_ticket
    tasks_from: resolve        # added in this phase
  vars:
    snow_ticket_close_notes: |
      Provisioned and patched by AAP.
        Instance:  {{ ec2_name }} / {{ ec2_instance_id }}
        Patched:   {{ patch_count | default(0) }} packages
        Rebooted:  {{ 'yes' if reboot_required | default(false) | bool else 'no' }}
        AAP job:   {{ aap_job_url | default('n/a') }}
```

## Done when

One launch produces one ticket whose work-note timeline matches the workflow graph, ending
resolved — and a running, patched instance tagged with that ticket number.

## Watch for

- Artifacts arrive as extra vars. Launch-time extra vars and survey answers have higher
  precedence — if a node sees a stale `snow_sys_id`, look for a survey field shadowing it.
- Namespace everything (`snow_*`, `ec2_*`, `patch_*`) so later additions don't collide.
- An `always` edge fires even when its parent failed — but only if the parent *ran*. See
  the note above before reaching for one.
- If the survey exposes `snow_sys_id`, make it optional and blank by default — otherwise
  [decision 1](architecture.md#1-make-node-1-skippable) silently inverts and node 1 stops
  creating tickets.
- Survey answers are launch-time extra vars for the **whole workflow**, so `ec2_name`,
  `ec2_instance_type` and `ec2_region` reach every node automatically. Only *discovered*
  values (instance id, IP, patch counts) need `set_stats`.

---

[← Phase 3](phase-3-patch-instance.md) · [Index](README.md) · [Phase 5 →](phase-5-failure-and-teardown.md)
