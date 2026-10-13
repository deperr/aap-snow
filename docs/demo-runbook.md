# Demo runbook — AAP + ServiceNow

For whoever is presenting. Build docs live in [mvp/](mvp/README.md); this is the script.

**The thing you are proving:** nobody opened this ticket, nobody updated it, nobody
closed it. The ticket is the audit trail and the automation wrote every line of it.

---

## 24 hours before

| ✅ | Why it matters |
|---|---|
| Log in to the ServiceNow PDI | PDIs hibernate after ~10 days idle and take several minutes to wake. This is the single most common way the demo dies. |
| Sync the AAP project | Picks up any commits since the last run. |
| Run `test-servicenow` | Proves the credential, the EE and the PDI in one job. |
| Run the full workflow once, clean | And **time it**. |
| Run the failure variant once | So you have seen the escalated ticket before the audience does. |
| Run `teardown` | Leave no instances running overnight. |
| Record a screen capture of a clean run | Your fallback if the network or AWS misbehaves live. |

**Check the patch count from your timed run.** If `dnf` found 0 packages the demo lands
flat — the ticket will say "Packages updated: 0". Pin an older image in the job
template's extra vars so there is something to install:

```yaml
create_vm_aws_image_filter: 'RHEL-9.4*HVM-*Hourly*'
```

---

## 15 minutes before

| ✅ | |
|---|---|
| Wake the PDI again | Confirm it responds, not just that it exists. |
| Confirm no leftover instance with your demo name | A name clash changes what node 2 does. |
| Pick a fresh hostname | `web-demo-01`, `web-demo-02`… Avoids clashes and keeps tickets distinct. |
| Open both screens, sized side by side | See below. |
| Close anything with customer names in it | Browser tabs, Slack, other ServiceNow records. |

### Screen layout

| Screen | Shows |
|---|---|
| **Left** | AAP — the workflow visualizer, zoomed so all six nodes are visible |
| **Right** | ServiceNow — the incident list, filtered to your service account, sorted newest first |

Have a third tab ready on the AWS EC2 console filtered by the `snow_incident` tag. You
will use it once, at the end.

---

## The run — roughly 8 minutes

### 0:00 · Frame it (45 seconds)

> "Two systems. On the left, Ansible Automation Platform. On the right, ServiceNow —
> a stock developer instance, nothing customised. I'm going to order a server. Watch
> the ticket, not the automation."

Do **not** explain the architecture yet. Let them see it work first.

### 0:45 · Launch

Launch the workflow. Fill the survey: hostname, blueprint `rhel9`, the rest defaults.

> "This is the standard Product Demos survey — same fields your team already uses."

### 1:00 · The ticket appears — the first beat

Refresh ServiceNow. The incident is there, In Progress, with the request details.

> "Nobody typed that. The automation opened it before it did any work — so if
> provisioning fails, there's still a record. That ordering matters more than it looks."

**Click into the ticket and stay there.** The rest of the demo happens in the activity
stream.

### 1:15 – 3:00 · Provisioning — the first wait

Node 2 runs for 90–150 seconds. This is your longest dead patch. Use it:

- Point out the workflow graph advancing on the left.
- "AAP is waiting for EC2 status checks, not just for the API to return — so the next
  node gets a host that's actually up."
- Explain the artifact mechanic now, while there's time: *"node one published the
  ticket's sys_id as a workflow artifact; every node downstream reads it. No database,
  no state file."*
- Take questions. You will get them here.

### 3:00 · Instance facts land — the second beat

Refresh. A second work note appears with instance ID, type, region/AZ, AMI, IPs.

> "Those are the real values, read back from AWS — not what I typed into the form."

### 3:00 – 5:00 · Patching — the second wait

- "It's applying every available update. Same play you'd run against a real fleet."
- Good moment for: *"everything you're watching is six playbooks and one role. The role
  exists so that when this becomes a change request instead of an incident, it's one
  file."*

### 5:00 · Patch results — the third beat

> "Forty-seven packages, reboot required. The ticket knows what happened to the box
> because the box told it."

### 5:00 – 6:30 · Reboot and verify

> "It's not just rebooting and hoping. It waits for the host to come back, then checks
> systemd actually reached a running state — a box that comes back degraded is the
> failure worth catching."

### 6:30 · Resolved — the payoff

The ticket moves to Resolved with the close notes.

> "Start to finish, nobody touched ServiceNow. The ticket is the audit trail and the
> automation wrote every line of it."

### 7:00 · Close the loop both ways

Switch to the AWS console tab. Show the instance's tags — `snow_incident` carries the
ticket number.

> "The ticket knows about the instance, and the instance knows about the ticket. If
> someone finds this box in six months, they can get back to why it exists."

---

## The failure run — do not skip this

This is what makes it credible. Budget two minutes.

**Setup:** relaunch with `create_vm_aws_securitygroup_name` set to a group with no SSH
ingress. Node 2 succeeds, node 3 fails at the SSH wait.

> "Same workflow, one thing broken. Watch what the ticket does."

**What to show:**

| | |
|---|---|
| The graph | Red node, failure edge routing to `snow-escalate` |
| The ticket | **Still open.** Urgency raised. A note saying how far it got. |
| The note | *"NOTE: instance i-… is still running and still costing money."* |

> "It didn't resolve a ticket for work that didn't finish, and it didn't quietly leave
> an orphan either — it told a human what's outstanding. Every path through this
> workflow ends with the ticket in a defined state: resolved, or escalated. Never
> ambiguous."

Why the SSH-failure variant and not a bad keypair: the instance **exists**, so the
escalation has something to report. A provisioning failure is a less interesting story.

---

## Questions you will get

| Question | Answer |
|---|---|
| "Can it be a change request instead of an incident?" | Yes. Every ticket write goes through one role; it's a module swap plus a different state model. Deliberate design choice. |
| "Can ServiceNow trigger it, rather than AAP?" | Yes — catalog item → Flow Designer → launch the workflow. The workflow already accepts an existing ticket's sys_id, so it adopts that ticket instead of opening one. |
| "Does it update the CMDB?" | Not yet. The CI payload is already assembled and published; it needs the API call. The real work there is doing it through the Identification and Reconciliation Engine so re-runs don't create duplicate CIs. |
| "Where are the credentials?" | AAP credentials, injected as environment variables. Nothing in the git repo reads a secrets file. |
| "Does this scale past one host?" | This demo is deliberately one instance, one ticket. Fleet patching is the same pattern with a dynamic inventory — that's the next conversation, not a limitation of the approach. |
| "Is this Product Demos?" | It uses the APD survey and tags, so it drops into an existing APD deployment. The ServiceNow integration is the new part. |
| "What if ServiceNow is down?" | The automation fails loudly rather than silently skipping the ticket. |

---

## If it breaks live

| Symptom | Do this |
|---|---|
| Ticket doesn't appear | PDI is probably asleep. Say so, switch to the recording. Don't debug on stage. |
| Node 3 hangs at the SSH wait | Security group or the instance has no public IP. Switch to the recording. |
| "Packages updated: 0" | Own it: *"this image is current — on a real estate you'd see dozens."* Move on. |
| Resolve fails with HTTP 400 | Close code isn't valid on that instance. Note it, move on; the demo's point is already made. |
| A node is just slow | Keep talking. The waits are in the script for a reason. |

**The rule:** one attempt at a live fix, then switch to the recording. An audience
forgives a backup video; they don't forgive ten minutes of someone reading error output.

---

## After

1. Run `teardown` — `-e create_vm_vm_name=<the name you used>`.
2. Confirm in the EC2 console that it's gone.
3. Resolve or delete the demo tickets if you want a clean board next time.

Leftover instances are the most likely way this demo generates an unwelcome invoice.
