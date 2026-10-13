# MVP: AAP + ServiceNow — EC2 Provision & Patch with a Self-Updating Ticket

**Status:** draft · **Owner:** Dennis Perrone · **Last updated:** 2026-10-13

## Goal

A single AAP workflow that:

1. **Provisions an EC2 instance.**
2. **Opens a ServiceNow ticket** carrying the real instance facts — hostname, instance ID,
   instance type, region/AZ, AMI, IP.
3. **Patches that same instance** in the same workflow run.
4. **Updates the ticket** with the patch results — how many packages, whether a reboot was
   needed — and resolves it.

The payoff line: *"Nobody opened this ticket, nobody updated it, nobody closed it. The
ticket is the audit trail and the automation wrote every line of it."*

### Non-goals for the MVP

- Production VPC/network design, golden AMIs, or Terraform-style state management.
- Patch compliance reporting beyond a work-note summary.
- Multi-instance / fleet patching. One instance, one ticket.

### Deliberately deferred, but on the roadmap

These are **not** discarded — they are the next two things this grows into, and
[the design decisions](architecture.md#design-decisions-that-keep-the-roadmap-cheap)
keep them cheap:

- **ServiceNow-triggered provisioning** ([Phase 6](phase-6-servicenow-trigger.md)). The MVP
  launches from AAP and creates tickets for us; eventually a catalog request in ServiceNow
  should drive the workflow.
- **CMDB registration** ([Phase 7](phase-7-cmdb.md)). Eventually the provisioned instance
  should land in the CMDB as a CI linked to the ticket, and patch state should update it.

## Demo narrative

~8 minutes, one story:

1. Operator launches **"Provision & Patch EC2"** with a survey: hostname, instance type,
   environment.
2. Ticket appears in ServiceNow immediately: *"Request received — provisioning
   `web-demo-01` (t3.micro, us-east-1)."*
3. EC2 instance comes up. The ticket updates with a block of **actual** instance facts —
   instance ID, AMI, private IP, AZ. The instance itself is tagged with the ticket number.
4. The workflow patches the instance. The ticket updates: *"Patching complete — 47
   packages updated, reboot required."*
5. Reboot, verify, and the ticket **resolves itself** with a link back to the AAP job.
6. Re-run with an injected failure. The ticket stays open, escalates to a human, and
   carries the error output as an attachment.

Step 6 is what makes it credible. Budget time for it.

## Documents

| Document | Contents |
|---|---|
| [architecture.md](architecture.md) | Workflow diagram, the two mechanics being proved, forward-compatibility decisions, repo layout |
| [aap-setup.md](aap-setup.md) | UI walkthrough: wiring the repo into the controller and running Phases 1-2 for real |
| [../demo-runbook.md](../demo-runbook.md) | Presenter script: pre-flight checks, the timed run, the failure run, likely questions |

### MVP phases

Each phase is independently demoable. Stop after any of them and you still have something
to show.

| Phase | Title | Effort |
|---|---|---|
| [0](phase-0-environment.md) | Environment bring-up and prerequisites | ~1 day |
| [1](phase-1-open-ticket.md) | Open the ticket from a playbook | ~½ day |
| [2](phase-2-provision-ec2.md) | Provision EC2, write real facts to the ticket | ~1 day |
| [3](phase-3-patch-instance.md) | Patch the instance, report results to the ticket | ~1 day |
| [4](phase-4-workflow.md) | Wire it into one AAP workflow | ~1 day |
| [5](phase-5-failure-and-teardown.md) | Failure path and cost control | ~½ day |

**MVP total: ~5 days.** Phase 0 carries the most schedule risk.

### Beyond the MVP

| Phase | Title | Effort |
|---|---|---|
| [6](phase-6-servicenow-trigger.md) | ServiceNow triggers the provisioning | ~2–3 days |
| [7](phase-7-cmdb.md) | CMDB registration and patch state | ~2 days |
| [8](phase-8-options.md) | Smaller options (change requests, approvals, dynamic inventory) | ~1 day each |

## Success criteria for the MVP

1. One workflow launch → exactly one ServiceNow incident and exactly one EC2 instance.
2. The ticket carries the real instance facts: hostname, instance ID, type, region/AZ,
   AMI, IP.
3. The ticket carries a patch summary: package count and reboot status.
4. A successful run resolves the ticket with a link back to the AAP job.
5. A failed run leaves the ticket open, escalated, and carrying the error output.
6. The instance is tagged with the ticket number — the loop closes in both directions.
7. End-to-end in under 5 minutes of wall clock (EC2 boot + `dnf` dominate; rehearse this).
8. Re-runnable back-to-back, with `teardown.yml` leaving nothing behind.

## Known risks

| Risk | Mitigation |
|---|---|
| Execution node can't SSH to the new instance | Resolve in [Phase 0](phase-0-environment.md). Same-VPC execution node is the reliable answer. |
| `dnf` update is slow or finds nothing to patch | Use a slightly older AMI; rehearse with a stopwatch. |
| PDI hibernation or reset | Wake it 24h before; keep `setup/` idempotent. |
| Hard-coded AMI goes stale | SSM parameter lookup, already in [Phase 2](phase-2-provision-ec2.md). |
| Live demo network dependency | Record a backup screen capture of a clean run. |
| Orphaned EC2 instances after rehearsals | `teardown.yml` + a tag-based reaper; run it every time ([Phase 5](phase-5-failure-and-teardown.md)). |
| `amazon.aws` parameter churn between versions | Pin the collection version in `requirements.yml`. |
| ServiceNow field validation surprises | Test each state transition in isolation in Phases 1–2. |

## Open questions

**Blocking the MVP:**

- **AWS account:** is there a sandbox account with a usable VPC, or does Phase 0 include
  building one?
- **Execution node placement:** can we run an execution node inside the demo VPC? This is
  the difference between a one-day Phase 0 and a three-day one.

**Shapes the roadmap, not the MVP:**

- **IntegrationHub entitlement.** Does the target environment have it? Decides whether
  [Phase 6](phase-6-servicenow-trigger.md) uses the Ansible spoke or a plain REST step.
  Worth finding out early even though it changes nothing today — the answer is often "no"
  and it's better to know.
- **Ticket type end-state.** If ServiceNow eventually drives this, the natural ticket is an
  RITM or change request, not an incident.
  [Decision 2](architecture.md#2-never-touch-the-ticket-outside-the-snow_ticket-role)
  keeps the swap cheap, but if the end state is already known, consider going straight to
  `change_request` in the MVP and skipping the migration.
- **CMDB ownership.** Is there a CMDB owner to satisfy? If yes,
  [Phase 7](phase-7-cmdb.md) should use the IRE path from day one rather than direct table
  writes.
