---
name: fsl-appointment-booking
description: "Return the bookable appointment slots a customer can choose for a service appointment, honoring operating hours and the scheduling policy. Use when a customer or agent needs to pick an arrival window before the appointment is scheduled."
---
# Goal:
Return the bookable time slots for one `ServiceAppointment` so a customer or agent can pick an arrival window. This is a **read** — it persists nothing. The chosen slot is handed to `fsl-schedule-appointment` to assign.

# Audience:
Booking agents and self-service flows offering arrival windows. Correct appointment, operating hours and time zone matter more than speed.

# Tool note — read via POST

`appointmentBooking` is a read, but the Field Service Connect API exposes it as **POST**, so it goes through **`dispatch`**, not `dispatch_readonly`. If only `dispatch_readonly` is available, say so and stop. Nothing here persists a change.

## 1. Resolve the appointment + operating hours

Resolve to exactly one `ServiceAppointment` Id and the operating-hours Id that defines bookable windows. Run both reads in one turn:

```
dispatch_readonly(method: "GET", url: "/services/data/v68.0/graphql",
  queryParams: { "queryInput": "{\"query\":\"query { uiapi { query { ServiceAppointment(where: { AppointmentNumber: { like: \\\"%[number]%\\\" } }, first: 25, orderBy: { LastModifiedDate: { order: DESC } }) { edges { node { Id AppointmentNumber { value } Subject { value } EarliestStartTime { value } DueDate { value } ServiceTerritory { Name { value } OperatingHours { Id { value } Name { value } } } } } } } } }\"}" })
```

```
dispatch_readonly(method: "GET", url: "/services/data/v68.0/graphql",
  queryParams: { "queryInput": "{\"query\":\"query { uiapi { query { OperatingHours(where: { Name: { like: \\\"%[hours]%\\\" } }, first: 10) { edges { node { Id Name { value } TimeZone { value } } } } } } }\"}" })
```

Prefer the territory's own `OperatingHours` from the first read. Fall back to the named lookup only if the appointment has none. **One appointment only** — resolve ambiguity by asking, never guessing.

## 2. Get bookable slots

POST through `dispatch`. `serviceId` is the appointment Id; `operatingHoursId` the resolved hours; `timeZone` from the operating hours; `exactAppointment` false for arrival windows, true for exact-start slots:

```
dispatch(method: "POST", url: "/services/data/v68.0/connect/field-service/scheduling-operations/appointmentBooking",
         body: { "serviceId": "[appointmentId]", "operatingHoursId": "[operatingHoursId]", "timeZone": "[timeZone]", "exactAppointment": false })
```

Returns candidate slots (window start/end, optionally a grade). Cite the values exactly.

## 3. Present the slots

```
## Bookable slots for [Appointment Number] — [Subject]
Territory: [Territory] · Time zone: [TimeZone]

| # | Date | Window |
|---|---|---|
| 1 | Mon 2026-11-24 | 08:00 – 10:00 |
| 2 | Mon 2026-11-24 | 13:00 – 15:00 |
| 3 | Tue 2026-11-25 | 08:00 – 10:00 |

No slots -> no availability in the appointment's window under current operating hours.
Pick a slot, then run fsl-schedule-appointment to assign it.
```

Zero slots → say so and name the constraint (window too tight, outside operating hours, no resource availability); do not widen the window yourself.
