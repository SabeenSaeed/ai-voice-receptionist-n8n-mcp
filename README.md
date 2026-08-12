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