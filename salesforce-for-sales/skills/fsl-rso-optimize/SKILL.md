---
name: fsl-rso-optimize
description: "Run a Field Service schedule optimization (RSO) over a territory or a single resource, preview the proposed schedule, and apply it only after explicit confirmation. Use to optimize a day or territory, rebalance a resource's route, or run Resource Schedule Optimization."
---
# Goal:
Run a Field Service optimization and apply it under control. **Stage a preview → review → confirm → apply → commit → verify.** `optimizeTerritory` / `optimizeResource` stage a preview; `applyPreview` applies it; `commitGanttUpdate` persists it. This skill **WRITES** to Salesforce — the preview and confirmation gate matter more than speed.

# Audience:
Dispatchers and scheduling admins running RSO (Resource Schedule Optimization), Global Optimization, or in-day optimization over a territory or one resource.

## 0. Write access

Needs the write-capable **`dispatch`** tool — every call here is POST through it. If only `dispatch_readonly` is available, say so and stop; optimization cannot run read-only.

## 1. Decide scope + resolve inputs

Two scopes:
- **Territory optimization** — a whole `ServiceTerritory` over a date range. Needs the territory Id, a scheduling policy Id, and start/end dates.
- **Single-resource optimization** — one `ServiceResource`'s route. Needs the resource Id, a policy Id, and a date (or range).

Resolve each by name to exactly one Id in one turn; ask on any ambiguity, never guess:

```
dispatch_readonly(method: "GET", url: "/services/data/v68.0/graphql",
  queryParams: { "queryInput": "{\"query\":\"query { uiapi { query { ServiceTerritory(where: { Name: { like: \\\"%[territory]%\\\" } }, first: 10) { edges { node { Id Name { value } } } } FSL__Scheduling_Policy__c(where: { Name: { like: \\\"%[policy]%\\\" } }, first: 10) { edges { node { Id Name { value } } } } ServiceResource(where: { Name: { like: \\\"%[resource]%\\\" } }, first: 10) { edges { node { Id Name { value } } } } } } }\"}" })
```

Confirm the scope, policy, and date range back to the user in one line before staging (optimization can move many appointments). The optimization type — RSO, GO (Global Optimization), or InDay — is carried as `originalRequestType` in step 3.

## 2. Stage the optimization preview

POST through `dispatch`. Staged only — persists nothing yet. Returns an `optimizationRequestId` (and a preview transaction id).

**Territory:**
```
dispatch(method: "POST", url: "/services/data/v68.0/connect/field-service/scheduling-operations/optimizeTerritory",
         body: { "territoryIds": ["[territoryId]"], "policyId": "[policyId]", "start": "[ISO start]", "finish": "[ISO finish]" })
```

**Single resource:**
```
dispatch(method: "POST", url: "/services/data/v68.0/connect/field-service/scheduling-operations/optimizeResource",
         body: { "resourceId": "[resourceId]", "policyId": "[policyId]", "start": "[ISO start]", "finish": "[ISO finish]" })
```

Keep the `optimizationRequestId` / preview transaction id — steps 3 and 4 need it. Where a body field is unspecified by the org's recipe, confirm the field with the user rather than guessing a value.

## 3. Review the preview, then confirm

Summarize what the preview changes — appointments moved, resources affected, projected travel or utilization delta if the preview returns it — and wait for an explicit yes:

```
## Optimization preview: [Territory or Resource]
Type: RSO · Policy: [Policy] · Window: 2026-11-24 → 2026-11-24

- Appointments rescheduled: 14
- Resources affected: 5
- Projected travel: 6.2h → 4.8h

Apply this optimization? (yes / cancel)
```

On "cancel", stop — the preview is discarded, nothing is applied.

## 4. Apply the preview (only on explicit yes)

`applyPreview` applies the staged optimization. `previewTxId` is the staged transaction; `originalRequestType` is the optimization type (`RSO`, `GO`, or `InDay`):

```
dispatch(method: "POST", url: "/services/data/v68.0/connect/field-service/scheduling-operations/applyPreview",
         body: { "previewTxId": "[preview transaction id]", "originalRequestType": "RSO" })
```

## 5. Commit

`commitGanttUpdate` persists the applied transaction. **Not reversible through this API** — only after the step-3 yes. Body carries the `apiPayload` from the apply/preview response:

```
dispatch(method: "POST", url: "/services/data/v68.0/connect/field-service/scheduling-operations/commitGanttUpdate",
         body: { "apiPayload": [payload from step 4] })
```

On failure at any step, report the exact error and which call produced it; do not retry with guessed values.

## 6. Verify

Re-read the affected appointments (by territory + window, or by the resource's assignments) and confirm the schedule changed, then output:

```
✅ Optimization applied: [Territory or Resource] (RSO)
- Appointments rescheduled: 14
- Window: 2026-11-24
Review the Gantt for the updated schedule.
```

If the committed result differs from the preview (concurrent changes, rule blocks), say exactly what came back.
