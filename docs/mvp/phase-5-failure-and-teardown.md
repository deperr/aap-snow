# Phase 5 — Failure path and cost control

[← Phase 4](phase-4-workflow.md) · [Index](README.md) · [Phase 6 →](phase-6-servicenow-trigger.md)

**Effort:** ~½ day · **Depends on:** [Phase 4](phase-4-workflow.md)

## Goal

The demo is not credible until failure is handled, and the repo is not safe to hand over
until instances get cleaned up. This phase completes the MVP.

## Steps

- Add a `fail_here` extra var that makes `patch-instance` deliberately fail.
- Wire an **on-failure** edge from each node to `snow-escalate`, which posts the failure
  summary as a work note, raises `urgency`, reassigns to a human assignment group, and
  attaches the failed task output with `servicenow.itsm.attachment_upload`. That
  attachment always gets a reaction from the room.
- On provisioning failure, decide explicitly: terminate the half-built instance, or tag it
  `status=quarantine` and leave it for inspection. Say which, in the work note.
- Add `playbooks/teardown.yml` that terminates anything tagged `managed_by=aap` in the demo
  VPC, and a `snow-close-ticket` variant that closes the ticket as part of teardown. Run it
  after every rehearsal.

## Done when

The failure run leaves an open, escalated, human-assigned ticket containing the error
text; the success run leaves a resolved one; and teardown reliably zeroes the AWS bill.

At this point the [MVP success criteria](README.md#success-criteria-for-the-mvp) should
all be met.

## Watch for

- ServiceNow requires **both** `close_code` and `close_notes` to move an incident to
  resolved/closed. Omitting either returns a validation error the module surfaces as a
  generic HTTP 400.
- Rehearse the failure run as often as the success run. It is the one that gets questions.
- Teardown is not optional housekeeping — orphaned instances are the most likely way this
  demo generates an unwelcome invoice.

---

[← Phase 4](phase-4-workflow.md) · [Index](README.md) · [Phase 6 →](phase-6-servicenow-trigger.md)
