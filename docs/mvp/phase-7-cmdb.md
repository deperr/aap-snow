> **Beyond the MVP.** Phases 6–8 are the direction of travel, not MVP scope. The MVP is
> built to accommodate them — see
> [design decisions](architecture.md#design-decisions-that-keep-the-roadmap-cheap).

# Phase 7 — CMDB registration and patch state

[← Phase 6](phase-6-servicenow-trigger.md) · [Index](README.md) · [Phase 8 →](phase-8-options.md)

**Effort:** ~2 days ·
**Depends on:** [decision 3](architecture.md#3-build-the-ci-fact-dict-now-even-though-nothing-consumes-it)

## Goal

The instance becomes a real CI, linked to the ticket, and patching updates it. Then the
CMDB feeds the *next* run's inventory — which is the moment the loop visibly closes.

## Steps

1. Create the CI after provisioning. The straightforward call:

   ```yaml
   - name: Register the instance as a CI
     servicenow.itsm.configuration_item:
       name: "{{ create_vm_vm_name }}"
       sys_class_name: cmdb_ci_linux_server    # see note on CI class below
       ip_address: "{{ ci_payload.ip_address }}"
       environment: "{{ ci_payload.environment }}"
       install_status: installed
       other:
         object_id: "{{ ci_payload.object_id }}"
     register: ci
   ```

2. Link the CI to the ticket by setting the incident's `cmdb_ci` field to
   `ci.record.sys_id`.
3. After patching, update the CI (a `u_last_patched` date field, or work-note equivalent
   if you don't want to modify the table).
4. Read it back: point an AAP inventory source at the `servicenow.itsm.now` inventory
   plugin, filtered to CIs with `managed_by=aap`. The next patch run gets its hosts *from
   the CMDB* — that is the screenshot worth building toward.

## Done when

A provisioning run creates (or reconciles to) exactly one CI, the ticket links to it, and
a second run against the same hostname does **not** create a duplicate.

## Watch for

- **Writing straight to `cmdb_ci_*` tables bypasses the Identification and Reconciliation
  Engine (IRE) and will create duplicate CIs** on re-runs. For anything beyond a demo, POST
  to `/api/now/identifyreconcile` (via `servicenow.itsm.api`) with a declared
  `discovery_source`, and let ServiceNow match-or-create. Raise this before a CMDB owner
  does — it is the first question any ServiceNow architect will ask, and knowing the
  answer buys a lot of credibility.
- Pick the CI class deliberately. `cmdb_ci_linux_server` exists everywhere; AWS-specific
  classes such as `cmdb_ci_ec2_instance` are more correct but depend on which CMDB/Cloud
  plugins the instance has. Verify on the PDI before writing playbooks against a class
  name.
- `discovery_source` must be one of the table's allowed choices. "Other Automated" is the
  usual pick for third-party automation.
- CMDB CRUD typically needs more than `itil` — budget time for the service account's roles.

---

[← Phase 6](phase-6-servicenow-trigger.md) · [Index](README.md) · [Phase 8 →](phase-8-options.md)
