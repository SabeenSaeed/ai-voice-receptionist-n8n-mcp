# Test Suite — Check Availability & Book Appointment Integration

**Date:** August 12, 2026
**Scope:** Verifies that Workflow 1 (Check Availability) and Workflow 2 
(Book Appointment) operate correctly as an integrated system against a 
live shared Google Calendar, with Airtable as the system of record for 
appointment metadata.

**Method:** Manual black-box testing via direct webhook invocation (curl), 
bypassing the VAPI voice layer to isolate backend logic. All test dates 
use August 2026 to align with clinic-hours edge cases (weekday/Friday/
Saturday/Sunday variations already in effect for that month).

---

## Results Summary

| # | Test Case | Category | Result |
|---|-----------|----------|--------|
| 1 | Full read-write-read cycle | Integration | ✅ Pass |
| 2 | Duration-aware overlap rejection | Conflict detection | ✅ Pass |
| 3 | Adjacent, non-overlapping booking | Boundary condition | ✅ Pass |
| 4 | Booking outside working hours | Business rules | ✅ Pass |
| 5 | Booking on a closed day (Sunday) | Business rules | ✅ Pass |
| 6 | Missing required field | Input validation | ✅ Pass |

**6/6 passing.**

---

## Test Detail

### 1. Full read-write-read cycle
**Goal:** Confirm both workflows operate against the same live calendar 
state, not cached or isolated data.

1. `POST /check-availability` for a clean date → returned full 40-slot day
2. `POST /book-appointment` for slot `10:00` (45 min service) → `success: true`
3. `POST /book-appointment` for slot `11:00` (45 min service) → `success: true`
4. `POST /check-availability` for the same date, re-queried →  
   `10:00`, `10:15`, `10:30` and `11:00`, `11:15`, `11:30` correctly excluded  
   from `available_slots`; `10:45` and `11:45` correctly remained available.

**Result:** Confirms duration-aware slot blocking, not naive exact-match 
filtering — a 45-minute appointment correctly consumes three 15-minute 
slot units.

### 2. Duration-aware overlap rejection
**Goal:** Confirm the booking workflow rejects requests that would overlap 
an existing appointment, even when the requested start time itself is not 
the exact start time of the conflicting event.

Existing appointment: `10:00–10:45`. Requested: `10:30` (would run 
`10:30–11:15`, overlapping the tail of the existing appointment).

**Result:** `success: false`. Confirms the conflict check evaluates full 
time ranges, not just start-time collisions.

### 3. Adjacent, non-overlapping booking
**Goal:** Confirm no off-by-one error causes valid boundary-adjacent slots 
to be incorrectly rejected.

Existing appointment ends `10:45`. Requested: `10:45` start.

**Result:** `success: true`. Confirms slot-boundary math is exact.

### 4. Booking outside working hours
**Goal:** Confirm clinic operating hours are enforced at booking time, 
not only at availability-check time.

Requested: `07:00` on a weekday (clinic opens `08:00`).

**Result:** `success: false`, generic slot-unavailable message. Confirms 
Book Appointment defers to Check Availability's business-hours logic 
rather than duplicating (and risking drift from) the ruleset.

### 5. Booking on a closed day
Requested: Sunday, any time.

**Result:** `success: false`. Consistent with test 4 — enforced via the 
same shared availability logic.

### 6. Missing required field
Requested: booking payload with `patient_name` present but empty.

**Result:** `success: false`, `"Missing required field(s): patient_name"`. 
Confirms input validation runs before any calendar or database write is 
attempted.

---

## Defects Found & Resolved During This Test Cycle

| Defect | Root Cause | Fix |
|--------|-----------|-----|
| Workflow executions completed in <100ms and returned stale data | Pinned/mocked data left active on multiple nodes from earlier manual testing, causing n8n to replay cached output instead of executing live API calls | Unpinned all nodes; added verification step to confirm execution duration is consistent with real API round-trip time |
| Successful bookings returned raw Airtable record objects instead of a formatted confirmation message | "Respond to Webhook" was bound to "Create a record" output directly, bypassing the message-formatting Code node | Rewired the graph so all terminal branches (success, validation error, calendar error, slot-unavailable) converge through their respective formatting nodes before reaching "Respond to Webhook" |
| Slot-conflict responses incorrectly returned `success: true` | The slot-unavailable branch and the success branch both fed into a shared downstream Code node that unconditionally rebuilt a success message from upstream booking data, ignoring which branch had triggered it | Rerouted the slot-unavailable branch to bypass the success-message builder entirely, connecting it directly to "Respond to Webhook", matching the pattern used by the other error branches |

---

## Known Limitations / Not Yet Tested

- Concurrent-request race conditions (two callers booking the same slot 
  within milliseconds of each other) — not testable via sequential curl 
  calls; would require a load-testing tool to simulate true concurrency.
- Dentist-specific filtering — deferred; current implementation treats 
  availability as clinic-wide across all three dentists sharing one calendar.
- Multi-day/recurring appointment patterns — out of scope for MVP.