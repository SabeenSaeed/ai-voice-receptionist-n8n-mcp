# Smile Forever Dental — Clinic Persona & Knowledge Base

This document is the single source of truth for the AI voice receptionist project.
Every n8n workflow, the VAPI system prompt, and the seed data in Airtable/Supabase
should reference this file so all parts of the system stay consistent.

Pricing and insurance data below are grounded in 2025 U.S. industry averages
(CareCredit, ADA practice data, and common PPO insurers), not invented figures —
sources noted at the bottom.

---

## 1. Clinic Basics

- **Name:** Smile Forever Dental
- **Address:** 4820 Maple Ridge Drive, Suite 200, Austin, TX 78731 *(fictional address — safe to reuse)*
- **Phone:** (will be the VAPI-provisioned number)
- **Hours:**
  - Monday–Thursday: 8:00 AM – 6:00 PM
  - Friday: 8:00 AM – 4:00 PM
  - Saturday: 9:00 AM – 1:00 PM (half-day — deliberate edge case for availability logic)
  - Sunday: Closed
- **Walk-ins:** Not accepted — appointment only, including emergencies (routed to the emergency line/escalation workflow)

---

## 2. Dentists

| Name | Title | Specialty | Works |
|---|---|---|---|
| Dr. Amelia Reyes | DDS | General Dentistry (cleanings, fillings, exams) | Mon–Fri |
| Dr. Farhan Malik | DMD | Orthodontics (braces, Invisalign consults) | Tue, Thu, Sat |
| Dr. Priya Nathan | DDS | Pediatric Dentistry | Mon, Wed, Fri |

This gives your "Check Availability" workflow a real filter parameter (`dentist` or `specialty`), not just a date lookup.

---

## 3. Services & Pricing

Prices reflect realistic 2025 U.S. general-dentistry averages (not clinic-specific negotiated/insurance rates).

| Service | Price Range (USD) | Typical Duration |
|---|---|---|
| Routine Check-up & Cleaning | $80 – $150 | 45 min |
| Dental X-Ray (set) | $50 – $120 | 15 min |
| Tooth Filling | $120 – $250 | 30–45 min |
| Root Canal Treatment | $500 – $1,200 | 60–90 min |
| Dental Crown | $800 – $1,500 | 2 visits |
| Tooth Extraction (simple) | $150 – $300 | 30 min |
| Teeth Whitening (in-office) | $300 – $650 | 60 min |
| Orthodontic Consultation | Free | 30 min |
| Braces (full treatment) | $3,000 – $7,000 | Multi-month plan |
| Invisalign (full treatment) | $3,500 – $8,000 | Multi-month plan |
| Pediatric Check-up & Cleaning | $60 – $120 | 30 min |
| Emergency Visit (pain/trauma) | $150 – $300 + treatment cost | Same-day priority slot |

*Receptionist rule: the AI should always give a price **range**, never a fixed quote, and should say final cost depends on the exam and insurance coverage — this is realistic and keeps the FAQ workflow safe from over-promising.*

---

## 4. Insurance Accepted

Modeled on the most commonly accepted PPO dental plans nationally:

- Delta Dental (PPO)
- Cigna Dental
- Aetna Dental
- MetLife Dental
- Guardian Dental
- UnitedHealthcare Dental

**Not accepted:** DHMO-only plans, out-of-state Medicaid dental plans.
*(Receptionist should say: "We accept most major PPO plans including Delta Dental, Cigna, Aetna, MetLife, Guardian, and UnitedHealthcare — I'd recommend confirming your specific plan with our front desk, or I can note it for the team to verify before your visit.")*

This "verify, don't guarantee" behavior is an important, realistic guardrail for the RAG/FAQ workflow — it avoids the AI making binding coverage claims.

---

## 5. Sample Patients (seed data for Airtable — Day 3)

| Name | Phone | DOB | Last Visit | Notes |
|---|---|---|---|---|
| Sarah Thompson | (512) 555-0142 | 1990-03-14 | 2026-02-10 | Routine cleaning, no issues |
| James Okafor | (512) 555-0198 | 1985-07-22 | 2025-11-03 | Crown placed on #14, follow-up due |
| Maria Gonzalez | (512) 555-0176 | 2002-01-09 | 2026-05-01 | Invisalign — month 4 of 14 |
| David Kim | (512) 555-0133 | 1978-11-30 | 2025-09-18 | Overdue for 6-month cleaning |
| Fatima Siddiqui | (512) 555-0165 | 1995-06-05 | 2026-04-22 | Whitening completed, satisfied |
| Ethan Brooks | (512) 555-0187 | 2015-08-19 | 2026-03-11 | Pediatric patient, Dr. Nathan |
| Lily Chen | (512) 555-0121 | 2018-02-27 | 2026-01-15 | Pediatric patient, mild anxiety — needs extra reassurance |
| Robert Hayes | (512) 555-0154 | 1965-04-11 | 2025-12-20 | Root canal #19, healing well |
| Aisha Khan | (512) 555-0109 | 1992-09-02 | 2026-06-01 | New patient, first cleaning scheduled |
| Michael Torres | (512) 555-0173 | 1988-12-15 | 2025-10-07 | Braces — Dr. Malik, month 8 |
| Grace Kelly | (512) 555-0161 | 1999-05-23 | 2026-02-28 | Extraction #32, no complications |
| Omar Farouk | (512) 555-0145 | 1980-01-17 | 2025-08-30 | Overdue, insurance on file: Delta Dental |
| Emily Ward | (512) 555-0192 | 2010-10-03 | 2026-05-19 | Pediatric, Dr. Nathan, sealants applied |
| Noah Patel | (512) 555-0116 | 1975-03-29 | 2025-07-14 | Crown consult scheduled |
| Sofia Ramirez | (512) 555-0184 | 1993-11-08 | 2026-06-10 | Whitening consult booked |

---

## 6. Emergency / Escalation Policy

The AI receptionist should escalate to a human (Slack/email alert) — not attempt to resolve — when the caller mentions:

- Severe or uncontrolled pain
- Facial swelling
- Trauma / knocked-out or broken tooth
- Uncontrolled bleeding
- Any request that sounds like a medical emergency unrelated to dental care (redirect to 911/ER)

The AI should **never give medical/dental advice** over the phone (e.g., "take ibuprofen," "it's probably a cavity"). It should acknowledge concern, gather basic info (name, callback number, symptom description), and hand off immediately.

---

## 7. Receptionist Persona

- **Name:** Ava
- **Role:** Virtual front-desk receptionist for Smile Forever Dental
- **Tone:** Warm, calm, efficient — like a well-trained real receptionist, not a chatbot. Short sentences. No medical jargon.
- **Boundaries:**
  - Never gives medical/dental advice
  - Never guarantees insurance coverage or exact pricing
  - Always confirms name + callback number early in the call
  - Escalates emergencies immediately rather than trying to "handle" them
- **Sample opening line:** "Thanks for calling Smile Forever Dental, this is Ava — how can I help you today?"

---

## Sources (pricing & insurance data)

- CareCredit — Dentist Prices: Dental Procedure Cost List
- Dental Clinic Price List 2025 (industry pricing survey)
- ParkSide Dental / Dental Plus Clinic — commonly accepted PPO insurers, 2025
- Delta Dental — 2025/2026 market position data

*Note: clinic name, address, dentists, and patients are entirely fictional and created for this portfolio project.*
