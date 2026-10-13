# AAP setup — running Phases 1 and 2 in the controller

[← Phase 0](phase-0-environment.md) · [Index](README.md)

A UI walkthrough for wiring this repo into AAP and running the first two phases against
your real ServiceNow PDI and AWS account. Work top to bottom; each step has a check
before you move on.

You create all credentials yourself — nothing in this repo reads a secrets file.

---

## 0. Prerequisite — the code must be in git

AAP pulls playbooks from a git project, so this repo needs to be pushed somewhere the
controller can reach (GitHub, GitLab, internal Gitea). A private repo is fine; you'll add
a Source Control credential for it.

**Check:** you can `git clone` the repo URL from a machine that isn't yours.

---

## 1. Values to have on hand

Collect these before you start — steps 4 and 7 need them.

| Value | Where it comes from | Example |
|---|---|---|
| ServiceNow instance URL | Your PDI | `https://dev123456.service-now.com` |
| ServiceNow service account | You create it; needs the `itil` role | `aap.integration` |
| AWS region | Your cloud env | `us-east-2` |
| Subnet ID | The demo VPC | `subnet-0abc…` |
| Security group ID | Must allow SSH **from the AAP execution node** | `sg-0abc…` |
| EC2 keypair name | Must exist in that region | `aap-demo-key` |
| Private key for that keypair | For the Machine credential | — |

> **Region note.** The repo's docs use `us-east-1` in examples; your local AWS config
> says `us-east-2`. Pick one and pass it as `ec2_region` consistently — the AMI lookup
> and the subnet/SG/keypair all have to agree.

**Do not demo as `admin`.** Create the `itil` service account now; someone will ask.

---

## 2. Execution environment

You have an EE with the ServiceNow collection, so select it on every job template in
step 7 rather than relying on the organisation default.

It needs three things:

| Needs | Why |
|---|---|
| `servicenow.itsm` | Phases 1-5 |
| `amazon.aws` | Phase 2 onward |
| `boto3` + `botocore` | Python deps for `amazon.aws` — **cannot** be installed from `requirements.yml` |

If your EE is missing `amazon.aws`, AAP can install it at job runtime from this repo's
`collections/requirements.yml` — but `boto3` must already be in the image either way.
`ee-supported-rhel9` includes all of it.

> **If you rely on `requirements.yml`, the Organization needs a Galaxy credential**
> (Administration → Organizations → your org → Galaxy Credentials, usually "Ansible
> Galaxy"). Without one, AAP skips the requirements file *without erroring*, and the job
> fails with a confusing "couldn't resolve module/action `servicenow.itsm.incident`".
> Not an issue if the EE already bundles everything.

**Check:** the smoke tests in step 5 import both collections successfully.

---

## 3. Project

Resources → Projects → Add.

| Field | Value |
|---|---|
| Name | `aap-snow` |
| Source Control Type | Git |
| Source Control URL | your repo URL |
| Source Control Branch | `main` |
| Source Control Credential | your SCM credential, if private |
| Options | tick **Update Revision on Launch** |

**Check:** the project syncs green.

---

## 4. The credential model

**No playbook in this repo contains a credential, a key, a username or an `instance:`
block.** Everything arrives as an environment variable injected by AAP at job runtime.
That is deliberate: it means the same playbooks run unchanged against your PDI, a
customer's instance, or a colleague's, with no edits.

| Credential | Type | What it injects | Used by |
|---|---|---|---|
| AWS | built-in *Amazon Web Services* | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SECURITY_TOKEN` | Phase 2 |
| ServiceNow | **custom** (step 4b) | `SN_HOST`, `SN_USERNAME`, `SN_PASSWORD` | Phases 1-5 |
| Machine | built-in *Machine* | SSH user + private key | Phase 3 |

Two rules that follow from this, and that the playbooks already obey:

- **Never set `ansible_user`.** The Machine credential supplies the login user. Setting
  it in a play, in `add_host`, or in `vars/demo.yml` silently overrides the credential.
  That is why `vars/demo.yml` has no SSH user in it.
- **`region` stays a playbook variable.** AAP's AWS credential injects keys but *not* a
  region, so `ec2_region` has to be passed in. This is the one piece of connection data
  that is legitimately a variable rather than a credential.

---

### 4a. AWS credential

Resources → Credentials → Add → Type **Amazon Web Services**. Access key + secret key.

Scope the IAM user tightly — `ec2:RunInstances`, `ec2:DescribeInstances`,
`ec2:DescribeImages`, `ec2:CreateTags`, `ec2:TerminateInstances`, and
`ssm:GetParameter` (the AMI lookup needs that last one, and it's the easy one to forget).

> **Leave "STS Token" blank; use long-lived IAM keys.** AAP's AWS credential injects the
> STS token as `AWS_SECURITY_TOKEN`, but `amazon.aws` **deprecated that variable in 6.0.0
> and removed it** — modern versions only read `AWS_SESSION_TOKEN`. With temporary
> credentials the token is therefore ignored and auth fails in a way that looks like a
> bad key. If you must use STS, add this to the job template's extra vars / environment
> so the modern name is set too, or pass `session_token` explicitly.

---

### 4b. ServiceNow credential — custom type

Check first whether your controller already ships a **ServiceNow ITSM** credential type
(Administration → Credential Types, filter by name). If it's there, use it and skip to
creating the credential. Otherwise create the type yourself.

#### Step 1 — create the credential type

Administration → **Credential Types** → **Add**.

- **Name:** `ServiceNow ITSM (basic auth)`
- **Description:** `Injects SN_HOST / SN_USERNAME / SN_PASSWORD for the servicenow.itsm collection`

**Input configuration:**

```yaml
fields:
  - id: SN_HOST
    type: string
    label: ServiceNow instance URL
    help_text: Full URL including scheme, e.g. https://dev123456.service-now.com
  - id: SN_USERNAME
    type: string
    label: Username
    help_text: Service account with the itil role. Do not use admin.
  - id: SN_PASSWORD
    type: string
    label: Password
    secret: true
required:
  - SN_HOST
  - SN_USERNAME
  - SN_PASSWORD
```

**Injector configuration:**

```yaml
env:
  SN_HOST: '{{ SN_HOST }}'
  SN_USERNAME: '{{ SN_USERNAME }}'
  SN_PASSWORD: '{{ SN_PASSWORD }}'
  SN_TIMEOUT: '60'
```

Save. The type is now reusable across every job template.

> **`SN_TIMEOUT` is not optional in practice.** The collection defaults to a **10 second**
> timeout, and a PDI routinely exceeds that on writes — especially in the first minutes
> after it wakes from hibernation. The failure looks alarming but is benign:
> `Unexpected error communicating with ServiceNow instance: The read operation timed out`,
> with `"timeout": 10.0` in `debug_info`. Setting it here fixes it for every job template
> at once. Raise it further if your instance is sluggish.

#### Step 2 — create the credential

Resources → **Credentials** → **Add**.

- **Name:** `ServiceNow PDI`
- **Organization:** your org
- **Credential Type:** `ServiceNow ITSM (basic auth)`
- Fill in the instance URL, the `itil` service account, and its password.

#### Step 3 — attach it

Add it to the `snow-open-ticket` and `provision-ec2` job templates (step 7). A job
template can hold multiple credentials of different types — `provision-ec2` needs both
this and the AWS one.

#### Why these three fields and no more

`servicenow.itsm` supports several auth modes, and they are **mutually exclusive in the
module's argument spec**: `username`, `api_key`, `access_token`, `refresh_token` and
`client_certificate_file` cannot be combined. A single credential type with optional
fields for several modes risks injecting an empty-but-present variable and tripping that
exclusion with a confusing error.

So: one type, one auth mode, all fields required. If you later need OAuth, create a
*separate* credential type rather than extending this one:

| Mode | Fields to inject |
|---|---|
| Basic (this one) | `SN_HOST`, `SN_USERNAME`, `SN_PASSWORD` |
| OAuth password grant | `SN_HOST`, `SN_USERNAME`, `SN_PASSWORD`, `SN_CLIENT_ID`, `SN_CLIENT_SECRET`, `SN_GRANT_TYPE=password` |
| API key | `SN_HOST`, `SN_API_KEY` |

> `validate_certs` and `api_path` have **no** environment variable in the collection — they
> can only be set as module parameters. Not an issue for a PDI, which has a valid
> certificate, but worth knowing if you ever point this at an instance behind a
> self-signed cert.

---

### 4c. Machine credential

Resources → Credentials → Add → Type **Machine**.

- **Username:** `ec2-user` (AL2023 and RHEL both)
- **SSH Private Key:** the private key matching the EC2 keypair from step 1

Not needed until Phase 3, but set it up while you're here.

Attach it only to the Phase 3 template — Phases 1 and 2 run entirely on `localhost` and
don't need it.

---

## 5. Smoke tests before anything else

These are [Phase 0](phase-0-environment.md) steps 4-6, and they exist to catch auth and
EE problems before the real playbooks confuse you. Run each as an ad-hoc command or a
throwaway job template against an inventory containing `localhost`.

**A — ServiceNow, end to end.** This repo ships the test as a playbook, so run it as a
job template rather than pasting YAML:

| Field | Value |
|---|---|
| Name | `test-servicenow` |
| Project | `aap-snow` |
| Playbook | `playbooks/test_servicenow.yml` |
| Inventory | Demo Inventory |
| Execution Environment | your EE with `servicenow.itsm` |
| Credentials | `ServiceNow PDI` only |

It checks credential injection, then read → create → update → resolve, which are exactly
the operations Phases 1-5 depend on. On success it prints a PASS block with the ticket
number and a direct link; on failure it prints the failed step and a table of likely
causes. Secrets are never printed either way.

The ticket is left **resolved** so you can eyeball it in ServiceNow. Options:

```
-e snow_test_caller=first.last   caller for the test ticket (default: admin)
-e snow_test_resolve=false       leave it open instead
-e snow_test_delete=true         delete it afterwards (usually needs admin, not itil)
```

If this passes, the plumbing is proven and any later failure is in the demo logic.

**B — AWS reachable, boto3 present:**

```yaml
- hosts: localhost
  gather_facts: false
  tasks:
    - amazon.aws.ec2_instance_info: {region: "<your region>"}
      register: r
    - debug: {msg: "{{ r.instances | length }} instances visible"}
```

**C — the one people skip.** Launch an instance by hand in the demo subnet, confirm the
**execution node** can SSH to it, then terminate it. A and B passing says nothing about
network path, and reachability is the single biggest risk in this demo. If your execution
node lives outside the VPC, settle that now, not during Phase 3.

---

## 6. Inventory

Any inventory containing `localhost` works — the stock **Demo Inventory** is fine.
Phases 1 and 2 both run `hosts: localhost`.

---

## 7. Job templates

### `snow-open-ticket` (Phase 1)

| Field | Value |
|---|---|
| Project | `aap-snow` |
| Playbook | `playbooks/snow_open_ticket.yml` |
| Inventory | Demo Inventory |
| Execution Environment | your EE with `servicenow.itsm` |
| Credentials | `ServiceNow PDI` only |
| Variables | see below |

```yaml
ec2_name: web-demo-01
ec2_instance_type: t3.micro
ec2_region: us-east-2
```

Tick **Prompt on launch** for Variables.

**Run it. Expected result:**

- A new incident in the PDI, state In Progress, with the opening work note.
- In the job's **Details → Artifacts** pane: `snow_sys_id` and `snow_number`.

If the artifacts pane is empty, the `set_stats` task didn't run — check the job ended
successfully rather than being cancelled.

### `provision-ec2` (Phase 2)

| Field | Value |
|---|---|
| Project | `aap-snow` |
| Playbook | `playbooks/provision_ec2.yml` |
| Inventory | Demo Inventory |
| Execution Environment | your EE (needs `amazon.aws` + `boto3` too) |
| Credentials | `ServiceNow PDI` **and** the AWS credential |
| Variables | see below |

```yaml
ec2_name: web-demo-01
ec2_instance_type: t3.micro
ec2_region: us-east-2
ec2_key_name: aap-demo-key
ec2_subnet_id: subnet-0abc...
ec2_security_group: sg-0abc...
```

This template needs `snow_sys_id` and `snow_number`, which come from Phase 1's artifacts
when the two run in a workflow. **To test it standalone**, add the values from the Phase 1
run to the variables above — the playbook asserts their presence and tells you so if
they're missing.

**Run it. Expected result:**

- A running instance tagged `Name`, `snow_incident`, `managed_by=aap`, `environment`.
- A second work note on the *same* incident listing instance ID, type, AZ, AMI and IP.
- Artifacts: `ec2_instance_id`, `ec2_private_ip`, `ec2_az`, `ec2_ami`, `ci_payload`.

---

## 8. Prove the artifact handoff

This is the mechanic the whole demo rests on, so test it explicitly rather than assuming.

Create a minimal workflow (Resources → Templates → Add → Workflow Job Template) with just
two nodes: `snow-open-ticket` → **on success** → `provision-ec2`. Launch it **without**
supplying `snow_sys_id` anywhere.

**Expected:** one incident, two work notes, one instance — and node 2 never had the
sys_id handed to it manually.

If node 2 fails its artifact assert, the artifact didn't cross. Check that node 1 actually
succeeded and that no survey field or workflow-level extra var is shadowing `snow_sys_id`.

---

## 9. When you're done

Terminate the instance. There's no `teardown.yml` yet — that lands in
[Phase 5](phase-5-failure-and-teardown.md). Until then, clean up by hand after every run,
or you will find a surprise on the bill.

---

## What this validates

Everything that local testing could not: that the EE actually carries the collections,
that credential injection works, that ServiceNow accepts the records, that the AMI
lookup resolves in your region, and that `set_stats` artifacts genuinely cross between
workflow nodes.

Report back what breaks and I'll fix it. The two most likely failures are the Galaxy
credential in step 2 and SSH reachability in step 5C.

---

[← Phase 0](phase-0-environment.md) · [Index](README.md)
