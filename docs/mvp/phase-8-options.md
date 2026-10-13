> **Beyond the MVP.** Phases 6–8 are the direction of travel, not MVP scope. The MVP is
> built to accommodate them — see
> [design decisions](architecture.md#design-decisions-that-keep-the-roadmap-cheap).

# Phase 8 — Smaller options

[← Phase 7](phase-7-cmdb.md) · [Index](README.md)

**Effort:** ~1 day each. Pick by audience; none depend on each other.

## Change request instead of incident

Provisioning is really a Standard Change. Swap `servicenow.itsm.incident` for
`servicenow.itsm.change_request` — mostly a module rename plus a different state model
(`new → assess → authorize → scheduled → implement → review → closed`). One file if
[decision 2](architecture.md#2-never-touch-the-ticket-outside-the-snow_ticket-role) held.

Strongest option for a change-management audience, and worth considering *before* the MVP
if that is the known end state.

## Approval gate before provisioning

An AAP *approval node* between nodes [1] and [2], or poll ServiceNow until the change
request reaches `scheduled`. "Automation waits for ITSM" lands well with governance-minded
rooms.

Coordinate with [Phase 6](phase-6-servicenow-trigger.md) — pick one approval point, not
two.

## Real dynamic inventory

Replace the `add_host` hop in [Phase 3](phase-3-patch-instance.md) with the
`amazon.aws.aws_ec2` inventory plugin filtered on `tag:managed_by=aap`, plus an *Inventory
Source Sync* node between [2] and [3]. More AAP-idiomatic than `add_host`, and the
stepping stone to fleet patching.

[Phase 7](phase-7-cmdb.md) step 4 then swaps the inventory source from AWS to the CMDB,
which is the more interesting version of the same change.

---

[← Phase 7](phase-7-cmdb.md) · [Index](README.md)
