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