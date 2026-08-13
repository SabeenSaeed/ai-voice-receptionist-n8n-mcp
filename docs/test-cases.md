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

---

# Test Suite — Reschedule / Cancel Appointment

**Date:** August 13, 2026
**Scope:** Verifies Workflow 3 in isolation and in combination with 
Workflow 1 (Check Availability), which it invokes as a sub-workflow to 
validate reschedule requests against live calendar state — the same 
pattern established in Workflow 2.

**Method:** Manual black-box testing via direct webhook invocation (curl) 
against the workflow's production endpoint. Testing initially used the 
single-use test webhook URL; this was abandoned mid-session in favor of 
the production URL after it was identified as the source of a false 
defect (see Defects table).

---

## Results Summary

| # | Test Case | Category | Result |
|---|-----------|----------|--------|
| 1 | Record not found (unregistered phone number) | Input handling | ✅ Pass |
| 2 | Cancel an existing booked appointment | Core flow | ✅ Pass |
| 3 | Reschedule to a new date/time | Core flow | ✅ Pass |
| 4 | Reschedule into an already-booked slot | Conflict detection | ✅ Pass |
| 5 | Reschedule to a valid, open slot (post-fix regression check) | Core flow | ✅ Pass |
| 6 | Unrecognized `action` value | Input validation | ✅ Pass |

**6/6 passing.**

---

## Test Detail

### 1. Record not found
Requested cancellation for a phone number with no `Booked` status record 
in Airtable.

**Result:** `success: false`, with a clear message inviting the caller to 
double-check the number or book a new appointment. Confirms the workflow 
distinguishes "no matching record" from a system error and responds 
accordingly, rather than failing silently or throwing a raw error.

### 2. Cancel an existing appointment
Booked a fresh appointment via Workflow 2, then issued a cancel request 
against the same phone number.

**Result:** Calendar event deleted, Airtable `Status` updated to 
`Cancelled`, and a formatted confirmation returned to the caller.

### 3. Reschedule to a new date/time
Booked a fresh appointment, then requested a reschedule to a different 
date and time with no conflicts.

**Result:** Original event deleted, new event created at the requested 
time, Airtable record updated with the new date/time while `Status` 
correctly remained `Booked` (not overwritten to `Cancelled`, confirming 
the two branches use independent update logic).

### 4. Reschedule into an already-booked slot
Booked two appointments for the same day at different times under 
different phone numbers, then attempted to reschedule the second into 
the first's exact slot.

**Result:** `success: false`, slot-unavailable message. Confirms 
reschedule requests are validated against live availability before any 
calendar mutation occurs — this is the most consequential test in the 
suite, as the initial build had no such check (see Defects).

### 5. Reschedule to a valid slot (regression check)
Re-ran the reschedule flow against an open slot immediately after fixing 
the availability-check defect, to confirm the fix didn't break the 
legitimate success path.

**Result:** `success: true`, correctly formatted confirmation.

### 6. Unrecognized action value
Sent a request with `"action": "delete_everything"` — a value matching 
neither of the workflow's two defined routes.

**Result:** `success: false`, with a message clarifying the two valid 
actions. Confirms the routing logic has an explicit fallback rather than 
failing open or silent.

---

## Defects Found & Resolved During This Test Cycle

| # | Defect | Root Cause | Fix |
|---|--------|-----------|-----|
| 1 | Zero-result search caused the entire execution to halt with no response | Airtable's "Search records" returns zero items (not an empty object) when no match exists; n8n halts downstream execution by default when a node produces no output | Enabled "Always Output Data" selectively, and confirmed the "If" node's condition correctly evaluates presence via array length rather than assuming a non-empty item always exists |
| 2 | Successful cancellations returned raw Airtable record objects instead of a caller-facing message | "Respond to Webhook" was bound directly to the Airtable update node's output, bypassing message formatting | Inserted a dedicated formatting node between the Airtable update and the response node, mirroring the pattern already established in Workflow 2 |
| 3 | Reschedule requests were accepted unconditionally, with no check that the new slot was actually free | The reschedule branch called Delete/Create directly with no validation step — an architectural gap, not a copy-paste error | Added a call to the Check Availability sub-workflow ahead of the delete/create sequence, gated by an `IF` node checking the requested slot against the returned `available_slots` array — identical pattern to Workflow 2's pre-booking validation |
| 4 | The Check Availability sub-workflow call and the calendar delete step both received `undefined` for date and event ID | Expressions referenced `$json.*`, which resolves to the *immediately preceding* node's output; several nodes upstream, `$json` no longer pointed at the record data those expressions assumed | Replaced ambient `$json` references with explicit node references (e.g. `$('Search records').first().json.fields.event_id`), which resolve correctly regardless of how many nodes sit in between |
| 5 | Manual testing intermittently produced "resource already deleted" errors on legitimate, single test attempts | Test-mode webhooks are single-use per arm; combining "Listen for test event" with the canvas "Execute workflow" button before VAPI was connected caused the same request to be processed twice against the same calendar resource | Switched manual testing to the workflow's permanent production webhook URL, removing the need to arm/re-arm and eliminating the double-execution path entirely |

---

## Design Notes

**Sub-workflow reuse for availability logic.** Both Workflow 2 (Book 
Appointment) and Workflow 3 (Reschedule) call Workflow 1 (Check 
Availability) rather than each implementing their own slot-conflict 
logic. This keeps business rules — clinic hours, per-dentist schedules, 
appointment-duration overlap — defined in exactly one place. A future 
change to clinic hours or slot granularity requires editing Workflow 1 
only; Workflows 2 and 3 inherit the change automatically.

**Known limitation — appointment lookup by phone number only.** 
"Search records" filters on `Phone` + `Status = 'Booked'` and does not 
disambiguate by date if a patient has more than one active booking. For 
this portfolio build, one active appointment per phone number is assumed. 
Production use would require the caller to specify which appointment 
(e.g. by date) when more than one match is returned.

---

## Workflow 4: Patient Lookup / CRM

**Test method:** Webhook integration tests via `curl` against
`/webhook-test/patient-lookup`, verified against live Airtable state.
**Result:** 14/14 passed

### Defects found and fixed during testing

| # | Defect | Root cause | Fix |
|---|---|---|---|
| 1 | Workflow halted silently on zero Airtable matches | `Search records` returns no items on a non-match; n8n does not propagate execution past a node with zero output items by default | Enabled **Always Output Data** on `Search records` |
| 2 | `"Welcome back, undefined..."` returned for a phone number with no real patient record | `IF: Patient Found?` only checked *record existence*, which is also true for placeholder ("New Patient") records | Added second condition, `IF: Real Patient?`, checking `Patient Name` / `Status` are not the placeholder value |
| 3 | Airtable node failed with `"DNS server returned an error"` | n8n Docker container was restarted without the `--dns` flags from the original `docker run` command; DNS config did not persist across container recreation | Recreated container with explicit `--dns 8.8.8.8 --dns 8.8.4.4` and `--restart unless-stopped`, so DNS configuration survives future restarts |

### Test cases

#### Valid, existing patient — format variance

| # | Input `phone` | Expected | Result |
|---|---|---|---|
| 1 | `512-555-7788` | `patient_found: true`, name `Production Test Patient` | ✅ Pass |
| 2 | `5125557788` | Same patient resolved; normalized to `512-555-7788` | ✅ Pass |
| 3 | `(512) 555-7788` | Same patient resolved | ✅ Pass |
| 4 | `+1 512-555-7788` | Same patient resolved; country code stripped | ✅ Pass |
| 5 | `+1 (512) 555-7788` | Same patient resolved; combined format handled | ✅ Pass |
| 6 | `5125557788` (as JSON number, not string) | Normalized and resolved correctly; confirms type coercion via `String()` | ✅ Pass |

#### Unknown caller / placeholder resolution

| # | Scenario | Expected | Result |
|---|---|---|---|
| 7 | First lookup of unregistered number `512-555-8899` | `patient_found: false`; placeholder record created in Airtable | ✅ Pass |
| 8 | Second lookup of the same number `512-555-8899` | Still `patient_found: false` (not a false-positive match on the placeholder record) — this is the case that exposed Defect #2 above | ✅ Pass |

#### Invalid input handling

| # | Input `phone` | Expected | Result |
|---|---|---|---|
| 9 | `512-555` (too short) | `error: INVALID_PHONE`, `retry_required: true` | ✅ Pass |
| 10 | `123` (far too short) | `error: INVALID_PHONE` | ✅ Pass |
| 11 | `512555778899` (too many digits) | `error: INVALID_PHONE` | ✅ Pass |
| 12 | `abc-def-ghij` (non-numeric) | `error: INVALID_PHONE` | ✅ Pass |
| 13 | `""` (empty string) | `error: PHONE_REQUIRED`, distinct message from `INVALID_PHONE` | ✅ Pass |
| 14 | `{}` (field omitted entirely) | `error: PHONE_REQUIRED` | ✅ Pass |
| 15 | `---( )---` (formatting characters only, no digits) | `error: INVALID_PHONE` | ✅ Pass |

### Coverage summary

- **Happy path:** existing patient resolution across 6 input format variants
- **Cold-start path:** unknown caller creation and correct non-match on repeat lookup
- **Negative path:** 7 distinct malformed/missing input cases, each returning the correct error code and caller-facing message
- **Infrastructure:** DNS/connectivity failure identified and permanently resolved at the container level, not worked around at the workflow level