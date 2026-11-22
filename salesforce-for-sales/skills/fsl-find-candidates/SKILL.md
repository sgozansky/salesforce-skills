---
name: fsl-find-candidates
description: "Score the Field Service resources that can take a service appointment, using the org's scheduling policy, and present the ranked candidates with their proposed times. Use to answer who can do this job, when, and how good is the fit before scheduling or optimizing."
---
# Goal:
Score the resources that can take one `ServiceAppointment` under a scheduling policy, and present the ranked candidates (resource, proposed window, travel, score). This is a **read** — it persists nothing. It is the first step of a fresh-scheduling flow and the input to `fsl-schedule-appointment`.

# Audience:
Dispatchers and schedulers choosing who to send. Correctness of the resolved appointment and policy matters more than speed.

# Tool note — read via POST

`getCandidates` is a read, but the Field Service Connect API exposes it as **POST**, so it goes through the **`dispatch`** tool, not `dispatch_readonly`. If only `dispatch_readonly` is available, say so plainly and stop — this operation cannot run read-only. Nothing in this skill persists a schedule change.

## 1. Resolve the appointment + policy

Reps give a Service Appointment number (`SA-nnnn`) or an account name, rarely an Id. Resolve to exactly one `ServiceAppointment` Id, and resolve the scheduling policy by name to its Id. Run both reads in one turn:

```
dispatch_readonly(method: "GET", url: "/services/data/v68.0/graphql",
  queryParams: { "queryInput": "{\"query\":\"query { uiapi { query { ServiceAppointment(where: { AppointmentNumber: { like: \\\"%[number]%\\\" } }, first: 25, orderBy: { LastModifiedDate: { order: DESC } }) { edges { node { Id AppointmentNumber { value } Subject { value } Status { value } EarliestStartTime { value } DueDate { value } ServiceTerritory { Name { value } } } } } } } }\"}" })
```

```
dispatch_readonly(method: "GET", url: "/services/data/v68.0/graphql",
  queryParams: { "queryInput": "{\"query\":\"query { uiapi { query { FSL__Scheduling_Policy__c(where: { Name: { like: \\\"%[policy]%\\\" } }, first: 10) { edges { node { Id Name { value } } } } } } }\"}" })
```

**Resolve to exactly one appointment before scoring.** No match → say so, don't guess. Multiple → list (number, subject, status, territory) and ask which. If the policy name is ambiguous or absent, ask which policy to use — never invent one.

## 2. Score the candidates

POST through `dispatch`. `serviceId` is the resolved appointment Id; `policyId` is the resolved policy Id:

```
dispatch(method: "POST", url: "/services/data/v68.0/connect/field-service/scheduling-operations/getCandidates",
         body: { "serviceId": "[appointmentId]", "policyId": "[policyId]" })
```

The response returns per-resource schedule data (candidate windows, travel, grade/score) and a `transactionId`. **Preserve the `transactionId`** — `fsl-schedule-appointment` reuses it so the assignment matches what was scored. Cite candidate values exactly as returned.

## 3. Present the ranked candidates

A ranked table, best fit first:

```
## Candidates for [Appointment Number] — [Subject] ([Territory])
Policy: [Policy Name]

| Rank | Resource | Proposed start | Proposed end | Travel | Score |
|---|---|---|---|---|---|
| 1 | Dana Cole | 2026-11-24 09:00 | 2026-11-24 11:00 | 18 min | 92 |
| 2 | Sam Ng   | 2026-11-24 13:30 | 2026-11-24 15:30 | 34 min | 78 |

No candidates -> the policy's rules (skills, availability, territory) exclude everyone in window.
To assign one of these, run fsl-schedule-appointment (reuses this scoring transaction).
```

If `getCandidates` returns zero candidates, say so and name the likely constraint (no matching skills, outside operating hours, no availability in the SA window) — do not widen the policy or window on your own.
