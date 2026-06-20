# Vocily Labs — Detailed Test Cases

How to read this:
- **Patient tests** = send a WhatsApp message to `918329914036` from a *different* phone (e.g. `8983340017`).
- **Staff tests** = post in the **"kushal labs group"** (`120363411817386712@g.us`) which contains the lab number.
- **Verify** = check the bot's reply on WhatsApp **and/or** the n8n execution + the Google Sheet.
- **Inspect an execution:** `GET /api/v1/executions?workflowId=LpnFG9kwWAdQOkiw&limit=5&includeData=true` (header `X-N8N-API-KEY`). Look at the `Brain` node output (`replyText`, `doBook`, `doReport`) and any `Append Row` / `Mark Ready` output.
- **Sheet check:** `GET /webhook/vocily-labs-lookup?phone=<number>` returns that patient's rows.

### Preconditions for a clean run
1. Inbound Router `LpnFG9kwWAdQOkiw` = **active**; WAHA session `default` = **WORKING**, webhook → `/webhook/vocily-labs-in`.
2. Google Sheets credential connected (re-auth if >7 days old).
3. Sheet reset to the 2 seeds (Rahul `8329914036` → Ready; Priya `8983340017` → In process) via the Setup workflow.
4. At least one working Gemini key in the rotation.

---

## A. Greeting & menu

| ID | Input (patient) | Expected | Verify |
|---|---|---|---|
| A1 | `hi` | Branded Kushal welcome menu (Book / Home collection / Report status / Timings). History resets. | Reply text = MENU |
| A2 | `Hello` / `namaste` / `menu` | Same menu (case-insensitive, synonyms). | Reply = MENU |
| A3 | `1` | "What's your full name?" (starts walk-in booking). | Brain enters booking ask |
| A4 | `4` | Timings + location + phones + Kushal package prices. | Reply = TIMINGS block |

---

## B. Walk-in booking

| ID | Input(s) | Expected | Verify |
|---|---|---|---|
| B1 (happy path, multi-turn) | `1` → `Ravi Kumar` → `CBC` | Each step prompts for the next; final: "✅ Booked! Token #N", Ref `VOC-YYYYMMDD-NNN`, Kushal branding. | Sheet: new row `Mode=Walk-in, Status=NEW, ReportStatus=Pending, Source=WhatsApp`, `Token`=N |
| B2 (natural, one line) | `I want to book a CBC test, my name is Ravi Kumar` | Books in one step (name+test extracted) → "✅ Booked! Token #N". | Row written, `Test=CBC`, `Name=Ravi Kumar` |
| B3 (natural, missing name) | `mujhe thyroid test karwana hai` | Asks "What's your name?"; after name → booked with Thyroid. | 2 execs; final row `Test≈Thyroid` |
| B4 (token increments) | Do B1 then another booking same day | Second booking gets `Token #N+1`, distinct `VOC_id`. | Two rows, distinct VOC_id/Token |
| B5 (no VOC collision with seeds) | Any booking today | `VOC_id` uses today's date → never overwrites yesterday-dated seeds. | Seeds unchanged |

---

## C. Home collection

| ID | Input(s) | Expected | Verify |
|---|---|---|---|
| C1 (explicit, with address) | `ghar pe sample lene aao, naam Sunil, thyroid, Ulkanagari` | "🏠 Home collection request received (Ref VOC-…). Team will call you." | Row `Mode=Home`, `Address=Ulkanagari`, `Name=Sunil` |
| C2 (address asked) | `home collection chahiye` → `Cidco, near X` | First asks for test+area, then captures. | Row `Mode=Home`, address set |
| C3 (NOT the lab's address) | `home collection for CBC` (no area) | Asks for the patient's address; must **not** auto-fill "CIDCO" (the lab's area). | Reply asks for address; no row until address given |
| C4 (walk-in stays walk-in) | `book a CBC test, name Ravi` | Routes to **Walk-in**, not Home (no home keyword). | Row `Mode=Walk-in` |

---

## D. Report status

| ID | Input (from) | Expected | Verify |
|---|---|---|---|
| D1 (in process) | `report ready hai kya?` from `8983340017` | "⏳ CBC — In process." | Lookup matches Priya |
| D2 (ready) | `report ready?` from `8329914036` | "✅ Thyroid Profile — Ready. Desk will share shortly." | Lookup matches Rahul |
| D3 (unknown number) | `report status` from a number not in the sheet | "Couldn't find a booking under this number — reply with your registered mobile number." | `found:false` path |
| D4 (number then lookup) | After D3 → reply `8983340017` | Looks up that number → status. | Gemini returns `phone`; lookup hits |
| D5 (last-10 match) | Lookup `918983340017` vs stored `8983340017` | Matches (last-10 normalisation). | Lookup returns Priya |

---

## DC. Cancellation

| ID | Sequence | Expected | Verify |
|---|---|---|---|
| DC1 (happy path) ✅verified | book a test (NEW) → `cancel my test` → `yes` | "Cancel your *CBC* (token #N)? Reply YES" → "✅ Your booking has been cancelled." | Sheet row `Status=Cancelled` |
| DC2 (decline) | …`cancel my test` → `no` | "👍 No problem — your booking is unchanged." | `Status` stays `NEW` |
| DC3 (nothing to cancel) | `cancel my booking` from a number with no NEW booking | "You don't have any open booking to cancel … please call us." | No sheet change |
| DC4 (already collected) | `cancel` when the only row is `Status=Collected` (e.g. a seed) | Treated as nothing-open → "call us" message (can't cancel a collected sample via bot). | Seed unchanged |
| DC5 (multiple open) | two open bookings → `cancel` → reply a token number → `yes` | Lists both with tokens, asks which; cancels only the chosen one. | Only that row `Cancelled` |
| DC6 (Hinglish) | `mera test cancel kar do` → `haan` | Same cancel flow in Hindi/Hinglish. | Works |
| DC7 (skipped downstream) | after cancel, `report ready?` for that number | Cancelled row not reported as pending; reminders cron skips it. | No status/reminder for cancelled |

---

## E. Price / FAQ / timings (no invented data)

| ID | Input | Expected | Verify |
|---|---|---|---|
| E1 | `thyroid test ka rate kya hai?` | Honest answer: individual prices not published → confirm at desk/phone; may mention packages. **No fabricated number.** | Reply contains phones / "confirm at desk", no invented ₹ |
| E2 | `what packages do you have?` | Lists Kushal's real packages (Diabetic ₹390 … Full Checkup ₹3,899). | Matches config |
| E3 | `kitne baje khulta hai?` | "7 AM – 11 PM, all days," + CIDCO location. | Timings answer |
| E4 | `do you do X-ray?` | Should say pathology-only / no imaging (not invent it). | No false "yes" |

---

## F. Conversation memory / context (the previously-broken cases)

| ID | Sequence | Expected | Verify |
|---|---|---|---|
| F1 (status then doubt) | `report ready?` → `but I never gave a sample` | Second message → **apology/clarification**, **NOT** a new booking. | Brain `action=answer`, no row |
| F2 (mid-flow detour) | `book a thyroid test` → `Abhishek` → `actually home collection instead` → `Ulkanagari` | Carries name+test, switches to home, books **once**. | Single Home row, Name+Test carried |
| F3 (time follow-up) | …(after a home request) `when will you visit me?` | Reassures "team will call to confirm" — **no menu dump, no loop**. | `action=answer` |
| F4 (preferred time) | …`11 am` | Acknowledges preferred time, ties to the pending request. | Context-aware answer |
| F5 (no duplicate booking) | A confused multi-message exchange | At most one booking row is created. | Count rows |

---

## G. After-hours capture (outside 7 AM – 11 PM IST)

| ID | Input (when lab closed) | Expected | Verify |
|---|---|---|---|
| G1 | `book a CBC test, name Ravi Kumar` at e.g. 1 AM IST | Booked **+** "🌙 we're closed, team will assist in the morning." | Row `Source = WhatsApp (after-hours)`; reply has the 🌙 note |
| G2 | Same booking during open hours | Normal confirmation, `Source = WhatsApp` (no 🌙 note). | Row `Source=WhatsApp` |
| G3 | `timings?` after-hours | Still answers normally (info is 24×7). | Timings reply |

---

## H. Report intake from staff group (status-only)

| ID | Staff posts (in group) | Expected | Verify |
|---|---|---|---|
| H1 (match) | `8983340017` (+ optional PDF) | Group: "✅ Marked Priya Patil … CBC as Ready and notified the patient." Patient `8983340017` texted "📄 your CBC report is ready." | Sheet: Priya `ReportStatus=Ready`, `report_uploaded_at` set; patient receives text |
| H2 (with test hint) | `8983340017\|CBC` | Same; `\|CBC` disambiguates if multiple open reports. | Correct row matched |
| H3 (unknown number) | `9999999999` | Group: "⚠️ No patient found for 9999999999." No patient texted. | `found:false`; no sheet change |
| H4 (no caption) | a PDF with **no** number | Group: "📄 post the patient's 10-digit number as the caption." | Prompt reply |
| H5 (other group ignored) | post in a different group the number is in | Bot ignores it (only `120363411817386712@g.us` is the staff group). | No execution action / `[]` |

---

## I. Edge cases & robustness

| ID | Scenario | Expected | Verify |
|---|---|---|---|
| I1 (@lid privacy id) | Real inbound where `from` = `…@lid` | Real phone resolved from `_data.key.remoteJidAlt`; replies reach the right person. | Reply delivered to correct number |
| I2 (Gemini one key 429) | A key in the rotation is throttled | Transparently uses the next key; no user-visible failure. | Reply still correct |
| I3 (Gemini all down) | All keys fail | Keyword fallback routes book/home/report/timings; else "could you rephrase / MENU." | No hard error; sensible reply |
| I4 (empty / sticker / no text) | message with no body | Ignored (`return []`), no crash. | Execution returns empty |
| I5 (fromMe) | the lab's own outgoing message | Ignored (no echo loop). | No reply |
| I6 (WAHA send fails) | transient WAHA error | `continueOnFail` → flow still records the row / marks ready. | Sheet still updated |
| I7 (onReceived no-retry) | WAHA inbound | n8n returns 200 instantly; no duplicate processing. | Single execution per message |

---

## J. Reminders cron (only when ACTIVATED)

| ID | Setup | Expected (within 15 min) | Verify |
|---|---|---|---|
| J1 (after-hours morning nudge) | an after-hours `NEW` row, lab now open, `Reminder` empty | Patient gets "🌅 Good morning, we're open — come in for your test." `Reminder` stamped. | Sheet `Reminder` set; one message |
| J2 (2-day follow-up) | a walk-in `NEW` row, `BookedAt` > 48 h, `Followup` empty | "your booking is still open, visit anytime." `Followup` stamped. | Sheet `Followup` set |
| J3 (dedup) | run cron again | No repeat message (stamped columns skip). | No duplicate sends |
| J4 (seeds skipped) | seed rows present | Seeds (`Source=Seed`, `Status=Collected`) never trigger a reminder. | No messages for seeds |

---

## K. Smoke test (fastest end-to-end demo, ~2 min)
1. Reset sheet (Setup). Confirm Rahul=Ready, Priya=In process.
2. From `8983340017`: `hi` → menu. ✔ (A1)
3. `report ready hai kya?` → "CBC — In process." ✔ (D1)
4. `mujhe ek cbc test book karna hai, naam Test User` → "✅ Token #1." Row appears. ✔ (B2)
5. In the staff group: post `8983340017` → you get "report ready" + Priya flips to Ready. ✔ (H1)
6. Reset sheet to restore the clean demo state.

---

## L. Image / PDF — does the lab conduct this test? (OCR)

*Executed 2026-06-21 via synthetic WAHA media webhooks (real fetch → Gemini-vision path). **0 wrong answers** — every non-happy case was dropped or safely fell back. Patient DMs only.*

| ID | Input | Expected | Result |
|----|-------|----------|--------|
| L1 | Photo of blood report (CBC, Lipid, Thyroid) | “✅ Yes, we do these” + book prompt | ✅ pass |
| L2 | Photo of scan report (USG, X-Ray, MRI) | “❌ scan/imaging — we’re a pathology lab” | ✅ pass |
| L3 | Mixed report (CBC + USG) | “✅ we do CBC; ❌ we don’t USG” | ✅ pass |
| L4 | Report photo **+ Hinglish caption** | availability answer (caption used as context) | ✅ pass |
| L5 | Serology (COVID RT-PCR, Dengue NS1, Widal) | “✅ yes” (classified pathology) | ✅ pass — *category-level, see DOC §8.7* |
| L6 | Vernacular (Devanagari) report | reads + answers | ⚠️ needs real test (tooling couldn’t render Devanagari) |
| L7 | Non-medical image (bill / sticker) | fallback “type the test name” | ✅ pass |
| L8 | **PDF** | same OCR path as image | ✅ fetch+parse proven (real medical PDF = on-device test) |
| L9 | Voice note / video | ignored — no reply, **no LLM call** | ✅ pass |
| L10 | Location / contact (no media) | ignored | ✅ pass |
| L11 | Unreachable / expired media URL | safe fallback, no crash | ✅ pass |
| L12 | Media in a **non-staff group** | ignored (gated) | ✅ pass |
| L13 | Our own outgoing (`fromMe`) | ignored | ✅ pass |

**On-device still to confirm:** real phone-photo OCR quality (blur/glare/angle), handwritten prescriptions, and vernacular (Devanagari) reports — send these to `918329914036`.

---
*Each test maps to the flows in `DOCUMENTATION.md` §6. If a test fails, pull the execution (`includeData=true`) and read the `Brain` node output first — it shows the parsed phone, the chosen action, and the reply.*
