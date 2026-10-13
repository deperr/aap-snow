# aap-snow

An AAP + ServiceNow demo: a workflow provisions an EC2 instance, opens a ServiceNow
ticket carrying the real instance facts, patches the instance, and updates and resolves
the ticket as it goes.

**Start here: [docs/mvp/](docs/mvp/README.md)** — goal, phased plan, and known risks.
**Running it in AAP: [docs/mvp/aap-setup.md](docs/mvp/aap-setup.md)**.

## The workflow

```mermaid
flowchart TD
    L(["Launch<br/><small>survey: hostname · instance type · env</small>"])

    N1["<b>1 · snow-open-ticket</b><br/><small>create the incident</small>"]
    N2["<b>2 · provision-ec2</b><br/><small>launch + tag the instance</small>"]
    N3["<b>3 · patch-instance</b><br/><small>dnf update</small>"]
    N4["<b>4 · reboot-and-verify</b><br/><small>reboot if required</small>"]
    N5["<b>5 · snow-close-ticket</b><br/><small>resolve</small>"]
    NE["<b>snow-escalate</b><br/><small>raise urgency, reassign,<br/>attach failure output</small>"]

    R(["Ticket resolved<br/>instance running + patched"])
    X(["Ticket open, escalated,<br/>assigned to a human"])

    L --> N1 --> N2 --> N3 --> N4 --> N5 --> R
    N2 -. "on failure" .-> NE
    N3 -. "on failure" .-> NE
    N4 -. "on failure" .-> NE
    NE --> X

    classDef snow fill:#e8f4ea,stroke:#2e7d4f,color:#1b3d2a
    classDef aws fill:#fdf1e3,stroke:#b8761b,color:#5c3a0a
    classDef fail fill:#fdeaea,stroke:#b03030,color:#5c1a1a
    classDef term fill:#eef1f6,stroke:#5b6882,color:#2b3243

    class N1,N5 snow
    class N2,N3,N4 aws
    class NE fail
    class L,R,X term
```

Every path ends in exactly one ticket-writing node, so the ticket always reaches a defined
state. Green nodes write to ServiceNow, amber nodes act on AWS. The detailed version
showing which artifacts flow along each edge is in
[docs/mvp/architecture.md](docs/mvp/architecture.md).

## Status

| Phase | | |
|---|---|---|
| [0](docs/mvp/phase-0-environment.md) | Environment bring-up | not started |
| [1](docs/mvp/phase-1-open-ticket.md) | Open the ticket from a playbook | **implemented**, not yet run against a live PDI |
| [2](docs/mvp/phase-2-provision-ec2.md) | Provision EC2, write facts to the ticket | **implemented**, not yet run against live AWS |
| [3](docs/mvp/phase-3-patch-instance.md) | Patch, report results to the ticket | not started |
| [4](docs/mvp/phase-4-workflow.md) | Wire into one AAP workflow | not started |
| [5](docs/mvp/phase-5-failure-and-teardown.md) | Failure path and cost control | not started |

## Layout

```
playbooks/     one playbook per workflow node
  test_servicenow.yml   standalone AAP <-> ServiceNow connectivity check
roles/
  snow_ticket/ every ServiceNow write goes through here
vars/demo.yml  shared demo defaults (overridable by survey / -e)
inventory/     localhost, for local testing
collections/   requirements.yml (installed collections are gitignored)
docs/mvp/      the plan
```

## Quick start

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

Variable names match the Ansible Product Demos **"Cloud | AWS | Create VM"** survey, so
that survey drives these job templates unchanged.

Credentials are read from the environment by the `servicenow.itsm` modules. In AAP they
come from the ServiceNow credential instead — nothing in this repo reads a secrets file,
and nothing should.

## Checks

```bash
ansible-lint playbooks/ roles/
yamllint playbooks/ roles/ inventory/
```
