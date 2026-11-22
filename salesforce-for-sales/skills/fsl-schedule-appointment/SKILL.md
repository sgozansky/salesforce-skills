---
name: fsl-schedule-appointment
description: "Assign a Field Service service appointment to a resource at a chosen time, checking scheduling-rule violations first and committing only after explicit confirmation. Use to book a specific resource onto an appointment from a candidate or chosen slot."
---
# Goal:
Assign one `ServiceAppointment` to a resource at a chosen start time. **Stage → check rules → confirm → commit → verify.** The `schedule` call stages the assignment; `commitGanttUpdate` persists it. This skill **WRITES** to Salesforce — the confirmation gate and rule check matter more than speed.

# Audience:
Dispatchers assigning a job. Pair with `fsl-find-candidates` (resource + score) or `fsl-appointment-booking` (customer slot) to pick the target.

## 0. Write access

Needs the write-capable **`dispatch`** tool — every call here is POST through it. If only `dispatch_readonly` is available, say so and stop; this skill cannot schedule read-only.

## 1. Gather the inputs

You need: the `ServiceAppointment` Id (`serviceId`), the target resource Id (`newResourceId`), the chosen `startDateTime` (ISO 8601), and the scheduling policy Id (`policyId`). If this follows `fsl-find-candidates`, reuse its resolved appointment, chosen candidate resource, proposed start, and policy. Otherwise resolve each by name first — one appointment, one resource, one policy; ask on any ambiguity, never guess. If `fsl-find-candidates` returned a `transactionId`, carry it so the assignment matches what was scored.

## 2. Stage the assignment

POST through `dispatch`. Staged only — nothing persists until step 4:

```
dispatch(method: "POST", url: "/services/data/v68.0/connect/field-service/scheduling-operations/schedule",
         body: { "serviceId": "[appointmentId]", "newResourceId": "[resourceId]", "startDateTime": "[ISO start]", "policyId": "[policyId]" })
```

Keep the staged transaction / `apiPayload` from the response — step 4 commits it.

## 3. Check rules, then propose and confirm

Before committing, run a lightweight rule check against the policy in session:

```
dispatch(method: "POST", url: "/services/data/v68.0/connect/field-service/scheduling-operations/checkRuleViolations",
         body: {})
```

Then show the staged assignment with any violations and wait for an explicit yes:

```
## Proposed assignment: [Appointment Number] — [Subject]
| Field | Value |
|---|---|
| Resource | Dana Cole |
| Start | 2026-11-24 09:00 |
| End (est.) | 2026-11-24 11:00 |
| Policy | Customer First |

⚠️ Rule violations: [list each, or "none"]

Commit this assignment? (yes / cancel)
```

Violations are warnings, not hard blocks unless the policy makes them hard — the dispatcher decides. On "cancel", stop without committing; the staged transaction is discarded.

## 4. Commit (only on explicit yes)

`commitGanttUpdate` persists the staged ESO transaction. **Not reversible through this API** — only run it after an explicit yes. The body carries the `apiPayload` from the staged `schedule` response:

```
dispatch(method: "POST", url: "/services/data/v68.0/connect/field-service/scheduling-operations/commitGanttUpdate",
         body: { "apiPayload": [staged payload from step 2] })
```

On failure (rule block, lock, stale transaction), report the exact error; do not retry with guessed values.

## 5. Verify

Re-read the appointment and its assignment, confirm it matches what was approved, and output:

```
✅ Scheduled [Appointment Number] — [Subject]
- Resource: Dana Cole
- Start: 2026-11-24 09:00
- Status: Scheduled
Record: https://[instanceUrl]/lightning/r/ServiceAppointment/[id]/view
```

```
dispatch_readonly(method: "GET", url: "/services/data/v68.0/graphql",
  queryParams: { "queryInput": "{\"query\":\"query { uiapi { query { ServiceAppointment(where: { Id: { eq: \\\"[id]\\\" } }) { edges { node { Id AppointmentNumber { value } Status { value } SchedStartTime { value } SchedEndTime { value } } } } AssignedResource(where: { ServiceAppointmentId: { eq: \\\"[id]\\\" } }) { edges { node { ServiceResource { Name { value } } } } } } } }\"}" })
```

If the saved values differ from what was approved (automation rewrote them), say exactly what came back.
