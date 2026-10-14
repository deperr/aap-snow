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
| [`vars/demo.yml`](../../vars/demo.yml) | Defaults and the blueprint table |

## Product Demos alignment

The variables match the APD **"Cloud | AWS | Create VM"** survey exactly, so that survey
can be attached to this job template unchanged. All twelve questions:

| Survey label | Variable | Notes |
|---|---|---|
| AWS Region | `create_vm_aws_region` | required |
| Name | `create_vm_vm_name` | required |
| Owner | `create_vm_vm_owner` | tag |
| Deployment | `create_vm_vm_deployment` | tag |
| Purpose | `create_vm_vm_purpose` | tag, defaults to `demo` |
| Environment | `create_vm_vm_environment` | `Dev` / `QA` / `Prod` |
| Blueprint | `vm_blueprint` | required |
| Subnet | `create_vm_aws_vpc_subnet_name` | a **name**, looked up to an id |
| Security Group | `create_vm_aws_securitygroup_name` | a **name** |
| SSH Keypair | `create_vm_aws_keypair_name` | required |
| AWS Instance Type | `create_vm_aws_instance_size` | default `t3.medium` |
| AWS Image Filter | `create_vm_aws_image_filter` | default `RHEL-9*HVM-*Hourly*` |

AMI discovery uses `ec2_ami_info` filtered by name/owner/architecture and sorted by
`creation_date`, as APD does — not an SSM lookup, which would ignore the survey's image
filter and cannot resolve RHEL images.

Tags match APD exactly (`Name`, `blueprint`, `environment`, `deployment`, `owner`,
`purpose`, `managed-by`, `apd`) plus `snow_incident`, so APD's dynamic inventory, VPC
report and teardown all see these instances.

`set_stats` publishes APD's `priv_network_private_ip` and `public_ip` alongside this
repo's `ec2_*` artifacts.

**Image and size are flat defaults** in [`vars/demo.yml`](../../vars/demo.yml), not a
blueprint lookup table:

```yaml
create_vm_aws_image_owners: '309956199498'   # Red Hat
create_vm_aws_instance_size: t3.medium
create_vm_aws_image_architecture: x86_64
create_vm_aws_image_filter: 'RHEL-9*HVM-*Hourly*'
```

Everything provisions RHEL 9 unless the survey overrides the size or filter.
`vm_blueprint` is now only a tag. The playbook still rejects Windows blueprints up
front, since Phase 3 patches with `dnf` and a RHEL box tagged `windows_full` would be
worse than a clear failure.

`create_vm_aws_image_owners` is quoted deliberately — unquoted it parses as an `int`,
and the module's `owners` parameter expects strings. It must never be blank: an
unowned search returns every public image matching the name across all AWS accounts,
and there is an assert enforcing that.

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
  -e create_vm_aws_region=us-east-2 \
  -e create_vm_vm_name=web-demo-01 \
  -e vm_blueprint=al2023 \
  -e create_vm_aws_vpc_subnet_name=aws-test-subnet \
  -e create_vm_aws_securitygroup_name=aws-test-sg \
  -e create_vm_aws_keypair_name=aws-test-key \
  -e snow_sys_id=<from phase 1> \
  -e snow_number=<from phase 1>
```

### Pinning the AMI

Set `create_vm_aws_instance_ami` and the whole discovery sequence — the `ec2_ami_info`
call, the match report, the assert and the sort — is skipped. Nothing else to change.

Worth doing for a rehearsal window: it removes the largest AWS call in the run, which
is the one that can stall an AAP job while the controller serialises its result. The
cost is that AMI ids are region-specific and go stale, so unpin it afterwards.

A **blank** value means "not pinned" and discovery still runs. That matters because AAP
sends an empty string for an untouched optional survey field, and the resolved id is
held in a separate variable (`ec2_ami_id`) rather than the input — `set_fact` loses to
an extra var, so reusing the input name would make a blank one impossible to fill in.

`-e create_vm_aws_subnet_id=subnet-…` similarly skips the subnet lookup.

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
- Variable parity with the live APD survey checked programmatically against
  `common/setup.yml` — all twelve present.
- Blueprint fallback asserted for every blueprint, for survey overrides, and for the
  blank-string case AAP actually sends for an untouched optional field.
- Empty `image_owners` (al2023) is dropped from the module call rather than leaking an
  `omit` placeholder.
- The missing-artifact guard fires with a usable message when `snow_sys_id` /
  `snow_number` are absent.
- Post-launch templating was unit-tested against a realistic `ec2_instance` return and
  against a degraded one (private subnet, no `public_ip_address`, no `cpu_options`) —
  both render, and `ci_payload` contains no `omit` placeholder.

**Not yet verified:** anything requiring live AWS — that the instance launches, that the
subnet and AMI lookups resolve against the APD stack, and that `state: started` returns
in a sensible time.

## Watch for

- **Subnet and security group are names, not ids.** The APD survey supplies
  `aws-test-subnet` / `aws-test-sg`. The subnet is resolved to an id with
  `ec2_vpc_subnet_info`; `security_group` accepts a name directly.
- **Never hard-code an AMI id.** They are region-specific and go stale — the single most
  common reason a cloud demo breaks three weeks later. Discovery picks the newest image
  matching the blueprint's filter.
- **`state: started`, not `running`.** `running` returns as soon as the instance is in the
  running state; `started` additionally waits for EC2 status checks to pass, which is a
  much better handoff to Phase 3. It is still *not* SSH-ready — status checks passing
  says nothing about sshd or key provisioning, so Phase 3 still needs
  `wait_for_connection`.
- **Idempotency has a sharp edge.** `ec2_instance` decides whether an instance already
  exists by filtering on Name tag, **base AMI**, state and subnet. AMI discovery returns
  whatever is newest *today*, so once a new image is published a re-run no longer matches
  and launches a **second** instance. The "expected 1 instance" assert catches this
  rather than letting it pass silently. Pin `create_vm_aws_instance_ami` across a
  rehearsal series if it becomes annoying.
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
