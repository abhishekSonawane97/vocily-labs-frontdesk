# Vocily Labs — Project Package

**WhatsApp Front Desk for Diagnostic Labs.** A patient messages the lab's WhatsApp in plain language; an AI books tests, captures home-collection requests, answers prices/timings, and reports when results are ready — 24×7, no human at the desk. Seeded for **Kushal Pathology Lab, Aurangabad**. Live demo number: **918329914036**.

## What's in this folder
| File | Contents |
|---|---|
| `DOCUMENTATION.md` | Full system doc — platform stack, architecture sketch, data model, the 4 workflows node-by-node, the AI Brain logic, every end-to-end flow (text sketches), how-to/runbook, decisions & limits, production path, ID/credential reference. |
| `TEST-CASES.md` | Detailed test cases (A–K) for every flow + edge cases + a 2-minute smoke test. |
| `workflows/` | Live JSON exports of all 4 n8n pipelines (re-importable). |

## The 4 pipelines (in `workflows/`)
| File | Workflow | Role | Status |
|---|---|---|---|
| `Vocily-Labs_Inbound-Router.json` | Inbound Router (`LpnFG9kwWAdQOkiw`) | The AI front desk — booking, home collection, report status, **cancellation**, FAQ, after-hours, staff report-intake. | ACTIVE |
| `Vocily-Labs_Lookup.json` | Lookup (`QWRu5KjmAes26om0`) | Match a phone → patient row (last-10 digits). | ACTIVE |
| `Vocily-Labs_Setup.json` | Setup (`lrftZj7qwGjCdJHj`) | Seed / reset the demo sheet. | OFF (utility) |
| `Vocily-Labs_Reminders.json` | Reminders (`ycLwHgDbI4qqnY8C`) | After-hours morning nudge + 2-day follow-up. | OFF (enable to use) |

## Stack
WhatsApp via **WAHA** (NOWEB, free) · logic + glue in **n8n** (self-hosted) · NLU via **Google Gemini 2.5 Flash** (4 rotating keys) · database = **Google Sheet** · production sending → **Meta WhatsApp Cloud API**. Hosted on GCP VM `ai-agent-server` (Dokploy).

## Quick start (demo)
1. Ensure Inbound Router is active + WAHA `default` session WORKING (webhook → `/webhook/vocily-labs-in`).
2. Reset the sheet via the Setup workflow (2 seeds: Rahul→Ready, Priya→In process).
3. From another phone, message **918329914036**: `hi`, then try `report ready hai kya?`, `book a cbc test`, `home collection chahiye`.
4. In the **kushal labs group**, post a patient's number → that patient gets "report ready."
5. See `TEST-CASES.md` §K for the full 2-minute smoke test.

## One known limitation
Free WAHA can't send a PDF document, so demo report delivery is the **text "your report is ready"** notification. The **actual PDF-in-chat is the production path via Meta Cloud API** (documented in `DOCUMENTATION.md` §8) — which is the production transport anyway.

---
*Full as-built record also at `../DIAGNOSTIC-LABS-DEMO.md` §8. Built 2026-06-17.*
