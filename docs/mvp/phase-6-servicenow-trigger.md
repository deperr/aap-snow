> **Beyond the MVP.** Phases 6–8 are the direction of travel, not MVP scope. The MVP is
> built to accommodate them — see
> [design decisions](architecture.md#design-decisions-that-keep-the-roadmap-cheap).

# Phase 6 — ServiceNow triggers the provisioning

[← Phase 5](phase-5-failure-and-teardown.md) · [Index](README.md) · [Phase 7 →](phase-7-cmdb.md)

**Effort:** ~2–3 days, mostly ServiceNow-side ·
**Depends on:** [decisions 1 and 2](architecture.md#design-decisions-that-keep-the-roadmap-cheap)

## Goal

Invert the demo: a user orders a server from the Service Catalog, and AAP fulfils it. The
MVP creates tickets for us; this makes ServiceNow the system of record that starts the
work.

## Flow

Catalog Item (variables: hostname, instance type, environment) → RITM created → Flow
Designer flow → launch the AAP workflow with `extra_vars` carrying the RITM's `sys_id` and
the catalog variables → the workflow adopts that ticket
([decision 1](architecture.md#1-make-node-1-skippable)) and narrates progress onto the
RITM instead of a new incident → on completion, the workflow sets the RITM's catalog task
to closed complete.

## Two ways to make the call, and the choice matters

| Approach | How | Catch |
|---|---|---|
| **Ansible Automation Platform spoke** (IntegrationHub) | Drag-and-drop action in Flow Designer; purpose-built for launching job templates. | **Requires an IntegrationHub subscription.** Frequently unavailable on a PDI and a common procurement blocker. Confirm entitlement *before* planning around it. |
| **Plain REST step** | Flow Designer REST step → `POST /api/v2/job_templates/{id}/launch/` with a bearer token, body `{"extra_vars": {...}}`. | Works on any instance including a PDI. Needs a ServiceNow Connection & Credential alias, and the job template must have `ask_variables_on_launch` enabled for the extra vars to take. |

Start with the REST step. It is unglamorous, it works everywhere, and it proves the
pattern; swap in the spoke later if the customer owns IntegrationHub.

## Done when

Ordering the catalog item provisions and patches an instance, with no AAP UI interaction,
and the RITM shows the full narration.

## Watch for

- `ask_variables_on_launch` (Prompt on launch → Variables) must be **on**, or AAP silently
  drops the inbound `extra_vars`. This is the most common failure in this phase.
- The launch endpoint returns immediately with a job ID — it does not wait. Either poll
  `/api/v2/jobs/{id}/` from the flow, or let the workflow write progress back to the RITM
  (which is the whole point, so prefer the latter and don't poll).
- Decide where approval lives: ServiceNow's catalog approval before the launch, or an AAP
  approval node mid-workflow. Both, and you will approve twice on stage.
- The ticket type changes from `incident` to `sc_req_item`. If
  [decision 2](architecture.md#2-never-touch-the-ticket-outside-the-snow_ticket-role) held,
  that is one file; if it didn't, budget an extra day.

---

[← Phase 5](phase-5-failure-and-teardown.md) · [Index](README.md) · [Phase 7 →](phase-7-cmdb.md)
