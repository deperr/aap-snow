# Phase 5 — Failure path and cost control

[← Phase 4](phase-4-workflow.md) · [Index](README.md) · [Phase 6 →](phase-6-servicenow-trigger.md)

**Effort:** ~½ day · **Depends on:** [Phase 4](phase-4-workflow.md)

**Status: implemented** (not yet run)

## Goal

The demo is not credible until failure is handled, and the repo is not safe to hand over
until instances get cleaned up. This phase completes the MVP.

## What was built

| File | Purpose |
|---|---|
| [`roles/snow_ticket/tasks/escalate.yml`](../../roles/snow_ticket/tasks/escalate.yml) | The `escalate` action — raises urgency, optionally reassigns, never changes state |
| [`playbooks/snow_escalate.yml`](../../playbooks/snow_escalate.yml) | The failure terminus |
| [`playbooks/teardown.yml`](../../playbooks/teardown.yml) | Terminates instances this demo created |

## One escalate node, every failure edge

A workflow node is not told which upstream node failed — but it does receive the
artifacts of the nodes that *succeeded*. The absence of an artifact is the signal:

| Missing artifact | Reported as |
|---|---|
| `ec2_instance_id` | stopped while provisioning the instance |
| `patch_count` | stopped while patching |
| `verify_systemd` | stopped while reboot or verification |
| none | stopped while closing the ticket |

So one node serves all three failure edges, rather than needing a separate node per
edge carrying its own `extra_data`.

`patch_count` is tested with `is not defined` rather than for emptiness: **0 packages
is a legitimate result** on a freshly built AMI, and a falsy-check would misreport a
later failure as a patching failure.

The note also flags when an instance was left running, because that one costs money.

## Teardown

Terminates the instance this run deployed. In a workflow it needs nothing —
`ec2_instance_id` arrives as an artifact from `provision_ec2.yml`:

```bash
# standalone
ansible-playbook playbooks/teardown.yml -e create_vm_vm_name=web-demo-01
```

Terminating by instance id is inherently scoped, so there is no bulk mode and no
confirmation prompt — it removes one instance, the one you named.

> **If you ever add a bulk mode, do not filter on the APD tags.** These instances carry
> `managed-by=aap-product-demos` and `apd=true` so APD's inventory and reports see
> them, which means those tags match **APD's entire cloud stack** — the Windows boxes,
> the report server, all of it. The name lookup here is additionally scoped with
> `tag-key: snow_incident`, which only this demo's instances carry.

## Wiring the failure edges

In the visualizer, add an **On Failure** edge from `provision-ec2`, `patch-instance`
and `reboot-and-verify` to a single `snow-escalate` node. Credentials: ServiceNow only.

Node 1 gets no failure edge — if it fails there is no ticket to escalate onto, and the
failure is visible in the job output.

## Injecting a failure to demo it

There is no deliberate failure switch; these are more honest and make better demos:

- Point `create_vm_aws_keypair_name` at a keypair that does not exist → node 2 fails
- Set `create_vm_aws_securitygroup_name` to one without SSH ingress → node 3 fails at
  the SSH wait, after the instance exists, which is the more interesting story

## Done when

The failure run leaves the ticket **open**, escalated and flagged with how far the run
got; the success run leaves it **resolved**; and teardown reliably zeroes the AWS bill.

At this point the [MVP success criteria](README.md#success-criteria-for-the-mvp) should
all be met.

## Watch for

- ServiceNow requires **both** `close_code` and `close_notes` to move an incident to
  resolved/closed. Omitting either returns a validation error the module surfaces as a
  generic HTTP 400. `snow_ticket/resolve` enforces both.
- Rehearse the failure run as often as the success run. It is the one that gets questions.
- Teardown is not optional housekeeping — orphaned instances are the most likely way this
  demo generates an unwelcome invoice.
- Attaching the failed job's stdout to the ticket would need the controller API and a
  controller credential, so the note links to the AAP job instead. A worthwhile addition
  if the audience asks for it.

---

[← Phase 4](phase-4-workflow.md) · [Index](README.md) · [Phase 6 →](phase-6-servicenow-trigger.md)
