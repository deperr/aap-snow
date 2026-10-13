# Phase 2 — Provision EC2 and write the real facts back to the ticket

[← Phase 1](phase-1-open-ticket.md) · [Index](README.md) · [Phase 3 →](phase-3-patch-instance.md)

**Effort:** ~1 day · **Depends on:** [Phase 1](phase-1-open-ticket.md) ·
**Status: implemented** (not yet run against live AWS or a live PDI)

## Goal

Launch the instance, tag it with the ticket number, and push the **actual** instance
facts — not the requested ones — onto the ticket.

## What was built

| File | Purpose |
|---|---|
| [`playbooks/provision_ec2.yml`](../../playbooks/provision_ec2.yml) | The Phase 2 playbook |
| [`vars/demo.yml`](../../vars/demo.yml) | Shared demo defaults (AMI parameter path, env, wait timeout, login user) |

The ticket update goes through `include_role: snow_ticket, tasks_from: note` rather than
calling `servicenow.itsm.incident` directly — that is
[decision 2](architecture.md#2-never-touch-the-ticket-outside-the-snow_ticket-role) doing
its job.

## Running it locally

Phase 2 consumes Phase 1's artifacts, so standalone runs must supply them by hand:

```bash
export SN_HOST=… SN_USERNAME=… SN_PASSWORD=…
export AWS_ACCESS_KEY_ID=… AWS_SECRET_ACCESS_KEY=…

ansible-playbook playbooks/provision_ec2.yml \
  -e ec2_name=web-demo-01 \
  -e ec2_instance_type=t3.micro \
  -e ec2_region=us-east-1 \
  -e ec2_key_name=my-demo-key \
  -e ec2_subnet_id=subnet-0123456789abcdef0 \
  -e ec2_security_group=sg-0123456789abcdef0 \
  -e snow_sys_id=<from phase 1> \
  -e snow_number=<from phase 1>
```

Add `-e ec2_ami_id=ami-…` to pin an AMI and skip the SSM lookup.

`boto3` and `botocore` must be present in whatever Python runs the modules. They ship in
the AAP execution environment; locally, `pip install boto3`.

## Done when

A running instance exists, tagged with the ticket number, and the ticket's activity
stream shows the instance-facts block.

## Verification status

Verified locally without AWS credentials or `boto3`:

- `ansible-lint` passes at the **production** profile; `yamllint` clean. Lint's `args`
  rule validates every `amazon.aws.ec2_instance` parameter against the installed
  argspec, so the parameter names are known-good.
- Both AMI branches behave: `ec2_ami_id` pins and skips the lookup; without it the SSM
  lookup runs.
- The missing-artifact guard fires with a usable message when `snow_sys_id` /
  `snow_number` are absent.
- Post-launch templating was unit-tested against a realistic `ec2_instance` return and
  against a degraded one (private subnet, no `public_ip_address`, no `cpu_options`) —
  both render, and `ci_payload` contains no `omit` placeholder.

**Not yet verified:** anything requiring live AWS — that the instance actually launches,
that the SSM parameter resolves in the target region, and that `state: started` returns
in a sensible time.

## Watch for

- **The SSM lookup is `amazon.aws.ssm_parameter`, not `amazon.aws.aws_ssm`.** The latter
  is a deprecated alias that still redirects, and most examples online (including the
  collection's own) still use it. Use the canonical name.
- Resolve the AMI by SSM parameter. Hard-coded AMI IDs are region-specific and go stale —
  this is the single most common reason a cloud demo breaks three weeks later.
- **`state: started`, not `running`.** `running` returns as soon as the instance is in the
  running state; `started` additionally waits for EC2 status checks to pass, which is a
  much better handoff to Phase 3. It is still *not* SSH-ready — status checks passing
  says nothing about sshd or key provisioning, so Phase 3 still needs
  `wait_for_connection`.
- **Idempotency has a sharp edge.** `ec2_instance` decides whether an instance already
  exists by filtering on Name tag, **base AMI**, state and subnet. The SSM lookup returns
  whatever the latest AL2023 AMI is *today*, so once Amazon publishes a new one, a re-run
  no longer matches the existing instance and launches a **second** one. The
  "expected 1 instance" assert catches this rather than letting it pass silently. Pin
  `ec2_ami_id` across a rehearsal series if it becomes annoying.
- **`purge_tags` defaults to `true`.** Because the playbook sets `tags`, any tag added
  out-of-band to the instance is removed on the next run.
- Put instance details in `description`/`work_notes` for the MVP. Custom `u_` fields look
  tidier but require a table change on the PDI — out of MVP scope.
- The `ci_payload` block is
  [decision 3](architecture.md#3-build-the-ci-fact-dict-now-even-though-nothing-consumes-it).
  It is dead weight today and saves a rewrite in [Phase 7](phase-7-cmdb.md). Note it uses
  `default('')` and not `default(omit)` — `omit` would serialise as a literal placeholder
  string inside `set_stats`.

---

[← Phase 1](phase-1-open-ticket.md) · [Index](README.md) · [Phase 3 →](phase-3-patch-instance.md)
