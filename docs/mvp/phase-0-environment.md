# Phase 0 — Environment bring-up and prerequisites

[← Architecture](architecture.md) · [Index](README.md) · [Phase 1 →](phase-1-open-ticket.md)

**Effort:** ~1 day · **Depends on:** nothing

Nothing here is demoable. Do it anyway — this phase exists to retire the risks that
otherwise surface halfway through Phase 3, when they are expensive.

## Components

| Component | Choice for MVP | Notes |
|---|---|---|
| ServiceNow | Personal Developer Instance (PDI) | Free. **Hibernates after ~10 days idle** — wake it the day before any demo. |
| Collections | `servicenow.itsm`, `amazon.aws`, `ansible.posix` | Pin versions in `collections/requirements.yml`. |
| AAP | 2.4+ / 2.5 controller | Needs workflow job templates and surveys. |
| AWS | One VPC, one subnet, one security group | SG must allow SSH **from the AAP execution node**, and the instance needs egress for `dnf`. |
| AMI | Amazon Linux 2023 or RHEL 9 | Resolve by SSM public parameter, not a hard-coded AMI ID. |
| Ticket type | `incident` | Simpler state machine than `change_request`. Swap in [Phase 8](phase-8-options.md). |

## Credentials to set up in AAP

- **AWS** — built-in *Amazon Web Services* credential type. Injects `AWS_ACCESS_KEY_ID`,
  `AWS_SECRET_ACCESS_KEY`, `AWS_SECURITY_TOKEN`. Scope the IAM user tightly:
  `ec2:RunInstances`, `ec2:DescribeInstances`, `ec2:CreateTags`, `ec2:TerminateInstances`.
  Someone will ask about blast radius.
- **Machine** — the SSH key matching the EC2 `key_name`, user `ec2-user` (AL2023) or
  `cloud-user`/`ec2-user` (RHEL). Needed by [Phase 3](phase-3-patch-instance.md).
- **ServiceNow** — newer controller versions ship a *ServiceNow ITSM* credential type. If
  yours does not, create a custom type:

  ```yaml
  # input configuration
  fields:
    - {id: SN_HOST,     type: string, label: Instance URL}
    - {id: SN_USERNAME, type: string, label: Username}
    - {id: SN_PASSWORD, type: string, label: Password, secret: true}
  required: [SN_HOST, SN_USERNAME, SN_PASSWORD]

  # injector configuration
  env:
    SN_HOST: '{{ SN_HOST }}'
    SN_USERNAME: '{{ SN_USERNAME }}'
    SN_PASSWORD: '{{ SN_PASSWORD }}'
  ```

  The `servicenow.itsm` modules read these env vars, so tasks need no `instance:` block.
  Use a PDI service account with `itil` — don't demo as `admin`.

## Steps

1. Wake/create the PDI, create the `itil` service account.
2. Create the AWS IAM user, VPC/subnet/SG, and upload the SSH keypair.
3. Create all three AAP credentials; create the project from this repo.
4. Smoke test A: `servicenow.itsm.incident_info` lists 5 incidents.
5. Smoke test B: `amazon.aws.ec2_instance_info` lists instances in the region.
6. Smoke test C: launch one instance by hand, confirm the execution node can SSH to it,
   then terminate it.
7. While that hand-launched instance is up, check whether `needs-restarting` exists on the
   AMI (it ships in `dnf-utils`/`yum-utils`). [Phase 3](phase-3-patch-instance.md) depends
   on the answer.

## Done when

All three smoke tests pass.

## Watch for

**Reachability is the #1 risk in this demo.** The AAP execution node must be able to SSH
to the new instance. Either run the execution node inside the same VPC, or give the
instance a public IP and allow the execution node's egress address in the SG. Settle this
here, not during Phase 3.

Smoke test C is the whole point of this phase. Do not skip it because the first two
passed — auth working says nothing about network path.

## Risk retired

Auth to both platforms, execution environment collection contents, and — most importantly
— network reachability from the execution node to a fresh EC2 instance.

---

[← Architecture](architecture.md) · [Index](README.md) · [Phase 1 →](phase-1-open-ticket.md)
