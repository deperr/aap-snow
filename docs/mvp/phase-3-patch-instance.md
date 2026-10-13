# Phase 3 — Patch the instance, report results to the ticket

[← Phase 2](phase-2-provision-ec2.md) · [Index](README.md) · [Phase 4 →](phase-4-workflow.md)

**Effort:** ~1 day · **Depends on:** [Phase 2](phase-2-provision-ec2.md)

**This is the core of the demo.** Everything before it is setup; everything after is
packaging.

## Goal

Patch the instance that was just provisioned, and have the ticket narrate it.

## Playbook

`playbooks/patch_instance.yml`:

```yaml
- name: Add the new instance to the in-memory inventory
  hosts: localhost
  gather_facts: false
  tasks:
    # No ansible_user here — the AAP Machine credential supplies the login user and
    # key. Setting it would override the credential. Locally, pass -u ec2-user.
    - ansible.builtin.add_host:
        name: "{{ ec2_private_ip }}"
        groups: new_instance

- name: Patch it
  hosts: new_instance
  gather_facts: false
  tasks:
    - name: Wait for SSH
      ansible.builtin.wait_for_connection:
        timeout: 300

    - name: Note that patching has begun
      servicenow.itsm.incident:
        sys_id: "{{ snow_sys_id }}"
        other:
          work_notes: "Instance reachable. Applying all available updates…"
      delegate_to: localhost          # ← see "Watch for" below

    - name: Apply all available updates
      ansible.builtin.dnf:
        name: '*'
        state: latest
      register: patch

    - name: Check whether a reboot is required
      ansible.builtin.command: needs-restarting -r
      register: restart_check
      changed_when: false
      failed_when: false

    - name: Publish patch outcome
      ansible.builtin.set_stats:
        data:
          patch_count: "{{ patch.results | default([]) | length }}"
          reboot_required: "{{ restart_check.rc != 0 }}"
        per_host: false

    - name: Report patch results to the ticket
      servicenow.itsm.incident:
        sys_id: "{{ snow_sys_id }}"
        other:
          work_notes: |
            Patching complete on {{ ec2_name }} ({{ ec2_instance_id }}).
              Packages updated: {{ patch.results | default([]) | length }}
              Reboot required:  {{ 'yes' if restart_check.rc != 0 else 'no' }}
      delegate_to: localhost
```

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
