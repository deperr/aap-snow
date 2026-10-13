# Phase 3 — Patch the instance, report results to the ticket

[← Phase 2](phase-2-provision-ec2.md) · [Index](README.md) · [Phase 4 →](phase-4-workflow.md)

**Effort:** ~1 day · **Depends on:** [Phase 2](phase-2-provision-ec2.md)

**This is the core of the demo.** Everything before it is setup; everything after is
packaging.

**Status: implemented** (not yet run against a live instance)

## What was built

| File | Purpose |
|---|---|
| [`playbooks/patch_instance.yml`](../../playbooks/patch_instance.yml) | The Phase 3 playbook |

Two plays: the first picks the address and does `add_host`, the second patches. Both
ticket writes go through `snow_ticket`, which already delegates to localhost and carries
the auth preflight.

## Connecting to the instance

The playbook prefers **`ec2_public_ip`** and falls back to `ec2_private_ip`, because the
AAP execution node usually sits outside the demo VPC. If yours is inside it, or the
instance has no public address, pass `ec2_connect_ip` explicitly.

The login user and key come from the AAP **Machine credential**. The playbook sets no
`ansible_user` anywhere — doing so would override the credential. Locally, pass
`-u ec2-user --private-key ...`.

## Goal

Patch the instance that was just provisioned, and have the ticket narrate it.

## What it reports

Two work notes. One when the host becomes reachable, naming the OS and kernel, and one
when patching finishes:

```
Patching complete on web-demo-01 (i-0a1b2c3d4e5f67890).
- Packages updated: 47
- Reboot required: yes
- Duration: 1m 52s
- OS: RedHat 9.6, kernel 5.14.0-570.el9.x86_64
```

Artifacts published for Phases 4-5: `patch_count`, `reboot_required`, `patch_seconds`.

## Done when

One run shows, in order on a single ticket: request received → instance facts → patching
started → patch results.

## Watch for

- **`delegate_to: localhost` on every ServiceNow task in a remote play.** Without it the
  module runs *on the EC2 instance*, which means the API call egresses from AWS and needs
  the collection present in the instance's Python environment. It will fail in a confusing
  way. This is the single most likely thing to trip you up in this phase.
- The `dnf` module's `results` is a list of changed-package strings; it is empty when
  nothing changed. A freshly launched AL2023 AMI may genuinely have ~0 updates — launch
  from a slightly older AMI if you want a satisfying number on screen.
- `needs-restarting` comes from `dnf-utils`/`yum-utils`. Install it first, or accept
  `rc != 0` meaning "not installed" rather than "reboot needed". You should already know
  which from [Phase 0](phase-0-environment.md) step 7.
- Keep ticket updates to ~5 per run. Each is a REST round-trip and the demo drags.
- `add_host` is deliberately crude here. Real dynamic inventory is
  [Phase 8](phase-8-options.md); it is not worth the setup cost until the pattern works.

---

[← Phase 2](phase-2-provision-ec2.md) · [Index](README.md) · [Phase 4 →](phase-4-workflow.md)
