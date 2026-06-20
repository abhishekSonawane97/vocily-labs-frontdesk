# Vocily Labs — WhatsApp Front Desk for Diagnostic Labs
### Complete System Documentation (as-built, 2026-06-17 · updated 2026-06-21: image/PDF test-availability)

---

## 0. TL;DR — what this is

A **WhatsApp "front desk"** for a diagnostic lab. A patient messages the lab's WhatsApp number in plain language (English / Hindi / Marathi / Hinglish); an **AI understands the intent** and the system books a test, captures a home-collection request, answers prices/timings, and tells the patient when their report is ready — 24×7, with no human at the desk. Lab staff mark a report "ready" by posting the patient's number in a private staff WhatsApp group. It can also read an **image or PDF of a report** a patient sends and answer one focused question — *does the lab conduct these tests?* — extracting only the test names (nothing else).

It is **not** a lab-management system. It is a **communication layer** that sits on top of whatever the lab already does. Everything is config-driven and seeded for **Kushal Pathology Lab, Aurangabad** — swap one config object to re-skin it for another lab.

**Demo number:** `918329914036` (WAHA) · **Seed lab:** Kushal Pathology · **Status:** live & working.

---

## 1. The platform stack — what each piece is and why it's used

| Platform | What it is | Why we use it | How it's used here |
|---|---|---|---|
| **WAHA** (WhatsApp HTTP API, `devlikeapro/waha`, **NOWEB** engine, CORE/free) | An unofficial bridge that turns a real WhatsApp account into an HTTP API (receive webhooks, send messages). | Zero onboarding, free, perfect for a **demo** on a number we control. | Receives every inbound message → posts it to n8n; n8n calls WAHA back to send replies. Session `default` linked to **918329914036**. |
| **n8n** (self-hosted workflow automation) | A visual + code workflow engine (nodes, webhooks, cron). Runs on the VM via Dokploy at `https://n8n.rohinisonawane.com`. | The "brain + glue." Holds the conversation logic, talks to Sheets/Gemini/WAHA, runs the cron. | 4 workflows (see §4). The core logic is one JavaScript **Code node** ("Brain"). |
| **Google Gemini 2.5 Flash** (LLM) | Google's fast, cheap large language model. | **Natural-language understanding** — so patients can type freely instead of pressing menu numbers. | The Brain sends the conversation to Gemini, which returns a JSON "decision" (intent + extracted fields + reply text). Multilingual. |
| **Google Sheets** | A spreadsheet, used as the live database. | A lab owner *understands a spreadsheet*. No DB, no app, no portal. | One sheet stores every booking/request + report status. n8n reads/writes it. |
| **Meta WhatsApp Cloud API** (official) | Meta's official WhatsApp business API. | **Production** sending — protects the client's number, sends PDFs natively, compliant. | **Not in the demo** — it is the documented production path for real PDF report delivery (see §8). |
| **Cal.com** | Open-source scheduling/booking. | Real appointment slots — only needed for **imaging** labs (radiologist must be present). | Not used for the blood-lab walk-in demo; reserved for the imaging variant (Panchandra). |
| **GCP VM + Dokploy** | The Ubuntu VM `ai-agent-server` (34.93.159.255) running Docker via Dokploy (Traefik, Postgres). | Hosts n8n + WAHA in one place, free of per-message platform fees. | n8n and WAHA are containers on the `dokploy-network`; n8n reaches WAHA at `http://waha:3000`. |

### Why this specific combination
- **WAHA + Gemini + Sheets + n8n = ~₹0 marginal cost** to run the demo, and ~₹150–400/mo per lab in production.
- The lab owner sees a familiar **WhatsApp chat** and a **Google Sheet** — nothing to learn.
- The "AI" (Gemini), the automation (n8n), the transport (WAHA/Cloud API) are all **invisible** to the owner. We sell a "WhatsApp Front Desk," not "an AI/n8n/WAHA stack."

---

## 2. High-level architecture (text sketch)

```
                                  PATIENT (any phone, natural language)
                                            │  WhatsApp
                                            ▼
                                   ┌──────────────────┐
                                   │   WAHA (NOWEB)    │  number 918329914036
                                   │   session=default │  host :3002 / internal http://waha:3000
                                   └────────┬─────────┘
                          inbound webhook   │   ▲  outbound: POST /api/sendText
                          (event=message)   │   │
                                            ▼   │
        https://n8n.rohinisonawane.com/webhook/vocily-labs-in
                                            │
                                   ┌────────▼─────────────────────────────────┐
                                   │  n8n: "Vocily-Labs Bot — Inbound Router"  │
                                   │                                           │
                                   │   BRAIN (one Code node)                   │
                                   │   • parse WAHA payload (@lid → real no.)  │
                                   │   • load session + history (static data)  │
                                   │   • is this the STAFF GROUP? ── yes ─┐     │
                                   │   • else: shortcuts (1/2/3/4, "hi")  │     │
                                   │   • else: ask Gemini (intent+fields) │     │
                                   └───┬───────────────┬─────────────┬────┘     │
                                       │               │             │          │
                            reply text │      doBook?  │   doReport? │          │
                                       ▼               ▼             ▼          │
                                ┌────────────┐  ┌────────────┐ ┌─────────────┐  │
                                │Send WhatsApp│  │ Append Row │ │Notify Patient│ │ (staff-group path)
                                │ (sendText)  │  │(upsert sheet)│ │+ Mark Ready │ │
                                └─────┬──────┘  └─────┬──────┘ └──────┬──────┘  │
                                      │ WAHA           │  Sheets        │ Sheets  │
                                      ▼                ▼                ▼         │
                                  patient        ┌─────────────────────────┐     │
                                                 │   GOOGLE SHEET (demo)    │◄────┘
                                                 │  1vown…vry5sY  (1 tab)   │
                                                 └───────────┬─────────────┘
                                                             │ read
                            ┌────────────────────────────────┼───────────────────────────┐
                            ▼                                 ▼                            ▼
                   ┌─────────────────┐              ┌──────────────────┐         ┌──────────────────┐
                   │ Vocily-Labs     │              │ Vocily-Labs      │         │ Vocily-Labs      │
                   │ Lookup (report  │              │ Reminders (cron, │         │ Setup (seeder/   │
                   │ status / match) │              │ 15 min, OFF)     │         │ reset, OFF)      │
                   └─────────────────┘              └──────────────────┘         └──────────────────┘

   GEMINI 2.5 Flash  ◄──── Brain calls it for NLU (key rotation over 4 keys)
```

---

## 3. Data model — the Google Sheet

- **Spreadsheet ID:** `1vown-S8-2cjs5r-YTLeR0bbOSMov-TwSfX4yhvry5sY`
- **Tab:** first tab, gid `0` (demo uses a single tab; daily-tab model is the production layout — see §8)
- **n8n credential:** Google Sheets OAuth2 "Google Sheets account" (id `BmsFNZFrUKr65DKi`)
- **Upsert key:** `VOC_id` (every write uses *append-or-update matching `VOC_id`*, so re-runs never duplicate)

### Columns (17)
| Column | Meaning |
|---|---|
| `VOC_id` | Unique internal id `VOC-YYYYMMDD-NNN` (date + daily serial). The match/upsert key. |
| `Token` | Patient-facing walk-in token number (the daily serial, e.g. `#3`). |
| `Name` | Patient name. |
| `Phone` | Patient number (stored as given; matching is by **last 10 digits**). |
| `Test` | Test / package requested. |
| `Mode` | `Walk-in` \| `Home` (\| `Imaging` in future). |
| `Address` | Home-collection address (+ preferred time note). Empty for walk-in. |
| `ApptISO` | Reserved for imaging/home preferred time. |
| `Status` | `NEW` → `Collected` → (`Cancelled`). |
| `ReportStatus` | `Pending` → `In process` → `Ready` → `Sent`. |
| `report_uploaded_at` | Timestamp when staff marked the report ready. |
| `report_sent_at` | Timestamp when the PDF was delivered (production / Cloud API). |
| `Source` | `WhatsApp` \| `WhatsApp (after-hours)` \| `Seed`. |
| `consent_flag` | `Y` — one-line consent (patient messaged us = opt-in). |
| `BookedAt` | ISO timestamp of the booking. |
| `Reminder` | Timestamp when the after-hours/morning reminder was sent (dedup). |
| `Followup` | Timestamp when the 2-day follow-up was sent (dedup). |

### Seed rows (demo)
| VOC_id | Name | Phone | Test | ReportStatus |
|---|---|---|---|---|
| `VOC-20260615-001` | Rahul Sharma | 8329914036 | Thyroid Profile (T3 T4 TSH) | **Ready** |
| `VOC-20260615-002` | Priya Patil | 8983340017 | CBC | **In process** |

*(Seeds are dated "yesterday" so the bot's "today" bookings never collide on `VOC_id`.)*

---

## 4. The four workflows (node-by-node)

### 4.1 `Vocily-Labs Bot — Inbound Router` (id `LpnFG9kwWAdQOkiw`, ACTIVE)
The patient-facing brain. Webhook receives **all** inbound WhatsApp (DMs *and* the staff group). It now also handles **incoming images and PDFs** (test-availability via OCR — see §5 step D and §6.9).

**Nodes & wiring**
```
Webhook (POST /webhook/vocily-labs-in, responseMode=onReceived)
   └─► Brain (Code)                         ← all logic; returns one item
         ├─► Send WhatsApp (HTTP→WAHA sendText)      uses {chatId, replyText}
         ├─► Booked?  (IF $json.doBook == true)
         │       └─► Make Row (Code: emit bookingRow) ─► Append Row (Sheets appendOrUpdate VOC_id)
         ├─► Report?  (IF $json.doReport == true)
         │       └─► Notify Patient (HTTP→WAHA sendText to patientChatId)
         │              └─► Mark Ready Row (Code) ─► Mark Ready (Sheets appendOrUpdate VOC_id)
         └─► Cancel?  (IF $json.doCancel == true)
                 └─► Make Cancel Row (Code) ─► Mark Cancelled (Sheets appendOrUpdate VOC_id → Status=Cancelled)
```
- **Webhook** — `onReceived` returns HTTP 200 instantly so WAHA never retries/duplicates.
- **Brain** — see §5 for the full logic.
- **Send WhatsApp** — `POST http://waha:3000/api/sendText`, header `X-Api-Key: <WAHA_API_KEY>`, body `{session:'default', chatId, text}`. `continueOnFail` so a send hiccup never breaks the flow.
- **Booked? → Make Row → Append Row** — writes a booking/home row (`appendOrUpdate` on `VOC_id`).
- **Report? → Notify Patient → Mark Ready Row → Mark Ready** — the staff-group path: text the patient + flip the row to `Ready`.

### 4.2 `Vocily-Labs Lookup` (id `QWRu5KjmAes26om0`, ACTIVE)
A tiny read API used by the Brain for report-status & report-intake matching.
```
Lookup Webhook (GET /webhook/vocily-labs-lookup?phone=…)
   └─► Read Sheet (Sheets, whole tab)
         └─► Filter (Code: match rows by LAST 10 DIGITS of phone) ─► JSON response
```
Returns `{found:true, VOC_id, Name, Phone, Test, Mode, Status, ReportStatus, report_sent_at, BookedAt}` per match, or `{found:false}`.

### 4.3 `Vocily-Labs Setup` (id `lrftZj7qwGjCdJHj`, DEACTIVATED)
Idempotent seeder / reset utility.
```
Webhook (POST /webhook/vocily-labs-setup)
   └─► Seed (Code: 2 seed rows) ─► Upsert (Sheets appendOrUpdate VOC_id)
```
Activate + `curl` the webhook to (re)seed. A "clear" variant (whole-sheet clear) was used during setup to wipe before reseed.

### 4.4 `Vocily-Labs Reminders` (id `ycLwHgDbI4qqnY8C`, DEACTIVATED)
Proactive nudges (off until you enable).
```
Every 15 min (Schedule)
   └─► Read Sheet
         └─► Find Due (Code)
               ├─► Send WhatsApp (sendText)
               └─► Clean (Code: strip to VOC_id+Reminder/Followup) ─► Mark Sent (Sheets upsert)
```
**Rules:** after-hours booking (`Source` contains "after-hours", `Status=NEW`, no `Reminder`) → once the lab is open, send *"Good morning, we're open — come in for your test"* and stamp `Reminder`. Any `Status=NEW` walk-in older than 48 h with no `Followup` → send a gentle *"booking still open"* nudge and stamp `Followup`. (Seeds are skipped.)

---

## 5. The Brain — full logic (the heart of the system)

The Brain is one async **Code** node. Order of operations:

**A. Parse the WAHA payload**
- Read `body.payload` from the webhook.
- `from` may be a privacy id `…@lid` (WAHA NOWEB). The **real phone** is in `payload._data.key.remoteJidAlt`. The Brain extracts digits, normalises (10-digit → prefix `91`), and builds `chatId = <number>@c.us`.
- Detect media: `payload.media` with mime `image/*` or `application/pdf` → `hasDoc`.
- Skip only if the message is from us (`fromMe`), or there is **neither text nor a usable image/PDF** — so media-only messages are no longer dropped.

**B. Session + memory (conversation context)**
- State lives in `$getWorkflowStaticData('global').sessions[<phone>]` — survives across messages.
- Each session keeps a **rolling history** of the last ~12 turns (`{role, text}`). This is what makes the bot *context-aware* (it understands "11 am", "actually home collection", "but I never tested" relative to the conversation).

**C. After-hours awareness**
- Computes the current **IST hour**; lab open = 07:00–23:00. If closed, booking confirmations get a *"we're closed, team will follow up in the morning"* note and the row is tagged `Source = WhatsApp (after-hours)`.

**D. Image / PDF — does the lab conduct this test? (OCR)** *(patient DMs only)*
- If the message carries an **image (`image/*`) or a PDF (`application/pdf`)**, the Brain fetches the file from WAHA (`media.url`, host rewritten to internal `http://waha:3000`; WAHA auto-downloads incoming media on the **free** Core tier, ~180 s lifetime), base64-encodes it, and sends it to **Gemini 2.5 Flash** inline (`inlineData`) with a strict prompt: *extract ONLY medical test/scan names and classify each `pathology | imaging | other`; return `{readable, tests[]}`; output nothing else.*
- **Category rule** (Kushal = pathology only): pathology → “✅ yes, we do it”; imaging/scan → “❌ we’re a pathology lab, we don’t do scans”; mixed → does/doesn’t list; only-other/ambiguous → “confirm at the desk”.
- **Confidence gate / privacy:** if `readable=false`, no test is found, or the fetch fails → a safe fallback (“please type the test name”). It **never** states values, ranges, diagnoses or any other content from the document, and the image is never stored.
- Non-document media (audio, video, stickers, location, contacts) is ignored with **no LLM call**.

**E. Staff-group report intake** (if the message is from group `120363411817386712@g.us`)
- Parse the patient number from the caption (`<number>` or `<number>|Test`).
- `Vocily-Labs Lookup` the number → if found, emit `doReport` with the patient's chat id + a "report ready" message + the `VOC_id`; the workflow then texts the patient, flips the row to `Ready`, and confirms to the group. If not found → reply to the group "no patient found."

**F. Deterministic fast-paths** (no LLM, instant)
- Greetings (`hi/hello/menu/namaste/…`) → branded menu (and reset history).
- Bare `1/2/3/4` → booking start / home / report-status / timings.

**G. AI dialogue manager** (everything else)
- Build a transcript of the conversation and ask **Gemini** for a strict-JSON decision:
  `{action, name, test, area, time, phone, reply}` where `action ∈ {ask, book_walkin, book_home, report_status, cancel, answer, menu}`.
- The prompt carries the **Kushal config** (hours, location, phones, packages) and hard rules: *default to walk-in*; *home only if the patient explicitly asks*; *never use the lab's address as the patient's*; *never invent prices/tests*.
- The Brain then **deterministically** executes the action — generates the token + `VOC_id`, writes the row, runs the lookup, etc. (Gemini decides + phrases; code does the writes — so a token is never hallucinated.)

**H. Gemini key rotation (reliability)**
- The dialog call rotates over **4 API keys** (working ones first). On a `429`/error it transparently tries the next key. Free-tier quota is ~100 calls/key/day, so 4 keys ≈ 400/day and it self-heals when one is throttled.

**I. Fallback (Gemini fully down)**
- Deterministic keyword routing (book/test → booking, home → home, report/ready → status, price/time → timings) + a graceful "could you rephrase / reply MENU" message. The bot never hard-fails.

---

## 6. End-to-end flows (text-sketch sequences)

### 6.1 Walk-in booking
```
Patient: "mujhe thyroid test karwana hai"
  → Brain → Gemini: {action: ask, test:"Thyroid"}  (needs name)
Bot:    "Sure! What's your name?"
Patient: "Abhishek"
  → Brain → Gemini: {action: book_walkin, name:"Abhishek", test:"Thyroid"}
  → Brain: token #N, VOC-YYYYMMDD-NNN; write row (Mode=Walk-in, Status=NEW)
Bot:    "✅ Booked! Token #N … show this at the desk …"
Sheet:  + row appears live
```

### 6.2 Home collection
```
Patient: "ghar pe sample lene aao, thyroid, Ulkanagari"
  → Gemini: {action: book_home, test:"Thyroid", area:"Ulkanagari"}
  → write row (Mode=Home, Address=Ulkanagari)
Bot:    "🏠 Home collection request received (Ref VOC-…). Our team will call you to confirm."
```
*(If the address is missing, the bot asks for it first, then books.)*

### 6.3 Report status
```
Patient: "report ready hai kya?"
  → Gemini: {action: report_status}
  → Lookup(sender's number) → Priya / CBC / In process
Bot:    "🔎 Your report status: ⏳ CBC — In process. We'll send it here as soon as it's ready."
```
*(If "Ready": "✅ … Ready. Our desk will share it shortly." If the number isn't found: asks for the registered mobile number.)*

### 6.4 Price / FAQ / timings
```
Patient: "thyroid test ka rate kya hai?"
  → Gemini answers from Kushal config (packages listed; individual prices not published → "confirm at the desk/phone")
Bot:    "<honest answer, no invented numbers>  …  Reply 1 to book."
```

### 6.5 After-hours capture (outside 7 AM–11 PM)
```
Patient (1 AM): "book a CBC test, name Ravi Kumar"
  → Brain knows lab is CLOSED → still books, tags Source="WhatsApp (after-hours)"
Bot:    "✅ Booked! Token #N …  🌙 Note: we're currently closed (open 7 AM–11 PM). Your request is saved — our team will assist you in the morning."
[later, if Reminders is ON] at 7 AM → "🌅 Good morning Ravi! We're open now — come in for your CBC (token #N)."
```

### 6.6 Report intake (staff → patient) — the highest-ROI piece
```
Lab staff (in "kushal labs group"): posts  "8983340017"  (+ a PDF, optional in demo)
  → Brain sees group msg → Lookup(8983340017) → Priya / CBC
  → flip row: ReportStatus=Ready, report_uploaded_at=now
  → Notify Patient (text): "📄 Hi Priya, your CBC report is ready! ✅ …"
  → Group confirm: "✅ Marked Priya Patil (8983340017) — CBC as Ready and notified the patient."
```
*In production, the same trigger also delivers the actual PDF via Meta Cloud API (§8).*

### 6.7 Reminders (cron, when enabled)
```
Every 15 min → read sheet → for each due row → send nudge → stamp Reminder/Followup (dedup)
  • after-hours booking + lab now open → "Good morning, we're open"
  • walk-in NEW > 48h, no follow-up    → "your booking is still open, visit anytime"
```

---

### 6.8 Cancellation
```
Patient: "cancel my test"  (or "booking cancel karna hai")
  → Gemini: action=cancel → Lookup(sender) → open (Status=NEW) bookings
  → one open: "Cancel your CBC (token #N)? Reply YES."   (Brain stores pendingCancel)
Patient: "yes"
  → Brain confirms deterministically → set row Status=Cancelled
Bot:    "✅ Your booking has been cancelled."
```
*Multiple open bookings → the bot lists them and asks which token. None → "no open booking to cancel; already-collected samples → call us." Cancelled rows are skipped by report-status and the reminders cron.*

---

### 6.9 Image / PDF — does the lab conduct this test? (OCR)
```
Patient sends a photo of a report / a PDF  (optionally with a caption)
     |  Brain: media is image/* or application/pdf
     v
fetch bytes from WAHA (media.url -> http://waha:3000)  ->  base64
     |
     v  Gemini 2.5 Flash (vision): extract test names + classify pathology|imaging|other
     |
     +- readable + pathology      -> "✅ Yes, we do <tests> — reply 1/2 to book"
     +- readable + imaging/scan   -> "❌ that's a scan; we are a pathology lab"
     +- readable + mixed          -> "✅ we do <...>;  ❌ we don't <...>"
     +- not readable / not a report / fetch failed -> "📄 please type the test name"
```
Scope: only **test-availability** is answered — never values, diagnosis or PII. Verified across 13+ cases (see `TEST-CASES.md` §L).

## 7. How-to / runbook (operations)

All API calls use the n8n base `https://n8n.rohinisonawane.com/api/v1` with header `X-N8N-API-KEY: <n8n key>`. WAHA calls run **on the VM** (`gcloud compute ssh ai-agent-server …`) against `http://localhost:3002` with header `X-Api-Key: <WAHA_API_KEY>`.

**Re-skin to another lab** — edit the `LAB={…}` config object + the Gemini prompt inside the Brain (name, area, phones, hours, packages) and the `STAFF_GROUP` id; redeploy the Inbound Router. Swap the demo sheet if you want a separate record store.

**Reset / re-seed the demo sheet**
```
# activate Setup, clear, seed, deactivate
POST /workflows/lrftZj7qwGjCdJHj/activate
POST https://n8n.rohinisonawane.com/webhook/vocily-labs-setup   # (clear variant then seed)
POST /workflows/lrftZj7qwGjCdJHj/deactivate
```

**Repoint WAHA to the labs bot (already done) / revert**
```
# point inbound at labs bot:
PUT  http://localhost:3002/api/sessions/default
     {"config":{"webhooks":[{"url":".../webhook/vocily-labs-in","events":["message"]}]}}
# revert (disable inbound):  set "webhooks":[]
```
*(Updating the config restarts the NOWEB session; it reconnects to WORKING without a QR re-scan because auth is stored.)*

**Reconnect the Google Sheets credential** (it expires ~weekly in OAuth "Testing" mode)
- n8n → Credentials → "Google Sheets account" → **Reconnect** (sign in with a Google account that can edit the sheet). Durable fix: publish that project's OAuth consent screen to "In production."

**Activate / deactivate reminders**
```
POST /workflows/ycLwHgDbI4qqnY8C/activate     # turn proactive nudges ON
POST /workflows/ycLwHgDbI4qqnY8C/deactivate   # OFF (default)
```

**Add / rotate a Gemini key** — edit the `GEM_KEYS=[…]` array in the Brain (working keys first), redeploy the Inbound Router.

**Update a workflow** — `PUT /workflows/<id>` with the full workflow JSON, then `POST /workflows/<id>/activate`. (The exports in `workflows/` are the current live versions.)

**Inspect what happened** — `GET /executions?workflowId=<id>&limit=N&includeData=true` shows each run's inbound payload, Brain output, and node results.

---

## 8. Key decisions & limitations

1. **WAHA NOWEB (free) cannot send a PDF document** — `sendFile` returns `422 "Plus only for NOWEB."` So **report delivery in the demo is status-only** (a text "your report is ready"). The **real PDF-in-chat is the production path via Meta WhatsApp Cloud API** (sends documents natively, free within the 24-h window) — which is the production transport anyway. *(Free alternatives if ever needed: WAHA WEBJS engine, or WAHA Plus license.)*
2. **Status-only report delivery chosen for the demo** (your decision) — zero cost/change; PDF via Cloud API in production.
3. **AI is a dialogue manager, never in control of writes** — Gemini decides intent, extracts fields, and writes the reply text; deterministic code generates tokens, writes the sheet, and runs lookups. Prevents hallucinated tokens/slots.
4. **Gemini key rotation** — free-tier quota (~100/key/day) is the only real constraint; 4 rotating keys + self-heal on 429. For high volume, enable billing on one key.
5. **Single tab for the demo** — the user's **daily-tab + monthly-merge** model is the *production* data layout; deferred because it touches every sheet-access node and the demo works cleanly on one tab.
6. **Proactive messages = mild WAHA ban risk** — the Reminders cron is OFF by default; in production these become Cloud API utility templates (compliant).
7. **Image/PDF answers use a *category* rule, not Kushal’s exact menu** — any blood/urine test classifies as “yes”. For a specific assay the lab does not run in-house it can still say “yes”; exact accuracy would need a test catalog (deferred — category rule chosen for zero maintenance).
8. **OCR readability is the swing factor** — clear printed reports read well; blurry / low-light / angled photos and handwritten prescriptions may not, and the bot then **safely falls back** to “type the test name”. Vernacular (Devanagari) reading is plausible but not yet validated on a real report.
9. **Receiving media is free** (WAHA Core, since v2024.10 — only *sending* files is Plus-gated), so the image/PDF feature adds **no cost** (it reuses the rotating Gemini keys).

---

## 9. Production path (what changes when a lab signs)

- **Sending → Meta WhatsApp Cloud API** (protects the lab's number; sends the report **PDF as a utility-template document**; free in-window; ~₹0.115/msg otherwise). ~30–60 min onboarding per lab, 250 recipients/24h before verification, verification typically 1–3 days.
- **Report intake split:** WAHA (or a Google Form) for the *internal* staff group; Cloud API for *patient-facing* sending (one number can't be on both; Cloud API can't read groups).
- **Daily-tab data model** for records (new tab/day, monthly merge to a master).
- **Per-lab config & sheet** (multi-tenant) + optional **imaging** Cal.com slot booking + (later, gated) **voice receptionist**.

---

## 10. Reference — IDs, endpoints, credentials

| Thing | Value |
|---|---|
| n8n base | `https://n8n.rohinisonawane.com` (API `…/api/v1`) |
| Inbound Router | id `LpnFG9kwWAdQOkiw` · `POST /webhook/vocily-labs-in` · ACTIVE |
| Lookup | id `QWRu5KjmAes26om0` · `GET /webhook/vocily-labs-lookup?phone=` · ACTIVE |
| Setup (seeder) | id `lrftZj7qwGjCdJHj` · `POST /webhook/vocily-labs-setup` · OFF |
| Reminders | id `ycLwHgDbI4qqnY8C` · 15-min cron · OFF |
| Demo sheet | `1vown-S8-2cjs5r-YTLeR0bbOSMov-TwSfX4yhvry5sY` (gid 0) |
| Sheets credential | "Google Sheets account" id `BmsFNZFrUKr65DKi` |
| WAHA | host `:3002`, internal `http://waha:3000`, key `<WAHA_API_KEY>`, session `default`, number `918329914036`, engine NOWEB |
| Staff group | `120363411817386712@g.us` ("kushal labs group") |
| Gemini | `…/v1beta/models/gemini-2.5-flash:generateContent?key=` · 4 rotating `AQ.Ab8RN6…` keys |
| VM | GCP `ai-agent-server`, zone `asia-south1-c`, project `project-92a34f99-5e61-4298-bb7` |

---
*Companion files in this folder: `workflows/` (live JSON exports of all 4 pipelines), `TEST-CASES.md` (detailed test scenarios), `README.md` (index).*
