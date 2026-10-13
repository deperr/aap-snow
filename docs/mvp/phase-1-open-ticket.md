# Phase 1 — Open the ticket from a playbook

[← Phase 0](phase-0-environment.md) · [Index](README.md) · [Phase 2 →](phase-2-provision-ec2.md)

**Effort:** ~½ day · **Depends on:** [Phase 0](phase-0-environment.md) ·
**Status: implemented** (not yet run against a live PDI)

## Goal

One playbook that creates the ticket and publishes its identity as a workflow artifact.
Everything downstream depends on that artifact, so get it right before adding cloud
complexity.

## What was built

| File | Purpose |
|---|---|
| [`playbooks/snow_open_ticket.yml`](../../playbooks/snow_open_ticket.yml) | The Phase 1 playbook |
| [`roles/snow_ticket/`](../../roles/snow_ticket/) | All ServiceNow writes live here ([decision 2](architecture.md#2-never-touch-the-ticket-outside-the-snow_ticket-role)) |
| [`roles/snow_ticket/tasks/open.yml`](../../roles/snow_ticket/tasks/open.yml) | Create-or-adopt the ticket |
| [`roles/snow_ticket/tasks/note.yml`](../../roles/snow_ticket/tasks/note.yml) | Append a work note (first used in Phase 2) |
| `ansible.cfg`, `collections/requirements.yml`, `inventory/hosts.yml`, `.ansible-lint` | Repo scaffolding |

`resolve` and `escalate` are deliberately absent — they belong to
[Phase 4](phase-4-workflow.md) and [Phase 5](phase-5-failure-and-teardown.md).

## Running it locally

```bash
ansible-galaxy collection install -r collections/requirements.yml -p ./collections

export SN_HOST='https://devXXXXXX.service-now.com'
export SN_USERNAME='aap.integration'
export SN_PASSWORD='…'

ansible-playbook playbooks/snow_open_ticket.yml \
  -e create_vm_vm_name=web-demo-01 \
  -e create_vm_aws_region=us-east-2 \
  -e vm_blueprint=al2023
```

In AAP these three variables come from the workflow survey
([Phase 4](phase-4-workflow.md)) and the credentials from the ServiceNow credential
([Phase 0](phase-0-environment.md)).

## Done when

The job template creates an incident in the PDI and the job's *Details → Artifacts* pane
shows `snow_sys_id` and `snow_number`.

## Verification status

Verified locally without a PDI:

- `ansible-lint` passes at the **production** profile; `yamllint` clean.
- Missing `create_vm_vm_name` / `create_vm_aws_region` / `vm_blueprint` fails on the
  assert with a usable message rather than a template error.
- With those set, execution reaches the `servicenow.itsm.incident` call and stops only on
  the absent `SN_HOST` — so everything up to the API boundary is exercised.
- The create/adopt branch logic was unit-tested against a truth table (undefined, empty,
  whitespace, null, real sys_id) and both identity-resolution branches.

**Not yet verified:** anything requiring a live PDI — that the incident is actually
created, that `caller` resolves, and that the artifacts surface in AAP. That is the first
task of a live Phase 1 run.

## Watch for

- `caller` is required when creating an incident, and expects a **user id**, usually
  `first_name.last_name`. The role defaults to `admin`; override `snow_ticket_caller`
  with the real service account.
- The module returns the record under `ticket.record`, not at the top level.
- `snow_sys_id` is normalised with `default('', true)`, not `default('')` — the
  two-argument form also catches an explicit `null`, which is what AAP sends when an
  extra var is declared with no value. Plain `default('')` only catches *undefined* and
  would leave the literal string `"None"`, silently flipping the role into adopt mode.
- Every ServiceNow task carries `delegate_to` + `run_once` inside the role, so callers
  can't forget once plays start targeting EC2 hosts in
  [Phase 3](phase-3-patch-instance.md).
- `ansible-lint` reports `args[module]: missing required arguments: host` on every
  `servicenow.itsm` task. It is a false positive — `host` is satisfied at runtime by the
  `SN_HOST` env fallback, which lint can't see in the static argspec. Suppressed per-task
  with `# noqa: args[module]` rather than repo-wide, so the rule still covers
  `amazon.aws` in Phase 2.

---

[← Phase 0](phase-0-environment.md) · [Index](README.md) · [Phase 2 →](phase-2-provision-ec2.md)
