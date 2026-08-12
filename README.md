# AI Voice Receptionist — VAPI + n8n + MCP

**Building in public**

An AI-powered voice receptionist for a dental clinic ("Smile Forever Dental"), 
using VAPI for real-time voice conversation on the front-end, and n8n on the 
back-end orchestrating 7 workflows through a native n8n MCP Server, handling 
appointment booking, rescheduling, patient lookup, FAQs (RAG), 
SMS/WhatsApp confirmations, and human escalation.

## Status
Day 1 of 15 (project scaffolding in progress).

## Stack
VAPI · n8n (self-hosted) · Groq (Llama) · Google Calendar · Airtable · 
Supabase (pgvector) · Twilio/WhatsApp Cloud API

## Progress log
- **Aug 10** Repo scaffolding, n8n instance setup, persona doc drafted

- **Aug 11** Connected Google Calendar via OAuth (self-hosted n8n on Docker/WSL2). 
  Built and fully tested Workflow 1 — Check Availability: webhook receives a 
  date, pulls events from the shared clinic calendar, and computes free 15-min 
  slots against real clinic hours (incl. Friday early close, Saturday half-day, 
  Sunday closed). Debugged Docker DNS instability, timezone-offset bugs in slot 
  comparison, and n8n's zero-output execution halt (fixed via "Always Output Data").

## Status
Day 2 of 15 — Workflow 1 (Check Availability) complete and tested.

## Workflows (MCP Tools)
- [x] 1. Check Availability
- [ ] 2. Book Appointment
- [ ] 3. Reschedule / Cancel Appointment
- [ ] 4. Patient Lookup (CRM)
- [ ] 5. FAQ / Knowledge Base (RAG)
- [ ] 6. SMS/WhatsApp Confirmation
- [ ] 7. Escalation / Human Handoff

## Progress log

- **Aug 12** Integrated Workflow 1 (Check Availability) and Workflow 2 (Book 
  Appointment) into a single verified pipeline. Book Appointment now calls 
  Check Availability as a sub-workflow to validate slot availability before 
  writing to the calendar, ensuring both workflows share a single source of 
  truth for scheduling logic rather than duplicating availability rules.

  Fixed three integration defects surfaced during testing: a data-pinning 
  artifact that caused executions to replay stale cached output instead of 
  live API calls; a webhook response node bound to the wrong upstream node, 
  causing raw Airtable records to leak into the API response instead of the 
  formatted confirmation message; and a branch-routing bug where slot-conflict 
  responses were being overwritten with success messages downstream.

  Verified the integration against a 6-case test suite covering the full 
  read-write-read cycle, duration-aware overlap detection, boundary-adjacent 
  bookings, clinic-hours enforcement, and input validation. Full results in 
  `docs/test-cases.md`.

## Workflows (MCP Tools)
- [x] 1. Check Availability
- [x] 2. Book Appointment — integrated with Workflow 1, tested (6/6 cases passing)
- [ ] 3. Reschedule / Cancel Appointment
- [ ] 4. Patient Lookup (CRM)
- [ ] 5. FAQ / Knowledge Base (RAG)
- [ ] 6. SMS/WhatsApp Confirmation
- [ ] 7. Escalation / Human Handoff

- **Aug 13** Built and fully verified Workflow 3 — Reschedule or Cancel 
  Appointment. The workflow locates a caller's active booking by phone 
  number, then branches on a `cancel` / `reschedule` action: cancellation 
  deletes the calendar event and marks the Airtable record `Cancelled`; 
  rescheduling re-validates the requested slot against Workflow 1 (Check 
  Availability) before deleting the old event and creating a new one, 
  ensuring reschedule requests are held to the same conflict-detection 
  rules as new bookings rather than bypassing them.

  Verified against a 6-case test suite covering record-not-found handling, 
  cancellation, rescheduling, slot-conflict rejection, and invalid input. 
  Found and fixed five defects during testing, the most significant being 
  a missing availability check on the reschedule path that would have 
  allowed a caller to reschedule directly into an already-booked slot. 
  Full defect log and test results in `docs/test-cases.md`.

  Also resolved a testing-workflow issue: manual re-arming of n8n's 
  single-use test webhook before every request was causing accidental 
  double-execution when combined with the canvas "Execute workflow" 
  button. Switched all manual testing to the permanent production webhook 
  URLs, eliminating the arm-per-request requirement entirely.

## Workflows (MCP Tools)
- [x] 1. Check Availability
- [x] 2. Book Appointment — integrated with Workflow 1, tested (6/6 cases passing)
- [x] 3. Reschedule / Cancel Appointment — integrated with Workflow 1, tested (6/6 cases passing)
- [ ] 4. Patient Lookup (CRM)
- [ ] 5. FAQ / Knowledge Base (RAG)
- [ ] 6. SMS/WhatsApp Confirmation
- [ ] 7. Escalation / Human Handoff