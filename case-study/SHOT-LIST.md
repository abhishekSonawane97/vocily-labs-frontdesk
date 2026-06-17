# Screenshot Shot-List & Export Guide

The decks (`deck-en.html` / `deck-hi.html` / `deck-mr.html`) ship **complete** — every chat is
recreated as a faithful WhatsApp-style bubble using the **real bot copy**. (`index.html` is the
language-chooser launcher.) Real screenshots are an **optional credibility upgrade** you can swap in.
This file tells you exactly what to capture, how to keep it anonymized, and how to export the PDF.

---

## A. Get the PDF (do this first — it works as-is)
**Easiest:** open the launcher `index.html`, pick a language, and click **⤓ Download PDF** — the
ready-made PDF downloads instantly.

**Or export fresh from a deck:**
1. Open a deck file (`deck-en.html` / `deck-hi.html` / `deck-mr.html`) in **Google Chrome**.
2. `Ctrl/Cmd + P` → **Print**.
3. Set: **Destination** = *Save as PDF* · **Layout** = *Landscape* · **Margins** = *None* ·
   **More settings → Background graphics = ON** (critical — otherwise colors print white).
4. Confirm the preview shows **13 clean pages**, nothing cut across a break. Save.

---

## B. Optional real screenshots (capture on the live demo: `918329914036`)
Capture from a **second phone**, then crop/relabel so the lab name reads neutrally
(**"City Pathology Lab"**, never the seed lab's real name). These map to three pages:

| Deck page | Send to 918329914036 | Expected reply to capture | Swaps in for |
|---|---|---|---|
| **6 · Patient Journey** | `mujhe thyroid test karwana hai` → then your name | "What's your name?" → "✅ Booked! Token #N …" | the booking chat |
| **8 · Report Delivery** | *(in the staff group)* post `8983340017` | patient gets "📄 your report is ready"; group shows "✅ Marked … Ready" | the report chat |
| **9 · Magic Moment** | after 11 PM IST: `Need a thyroid test tomorrow morning` | "…request is saved … we're closed (open 7 AM) … 🌙" | the after-hours chat |

**Anonymizing a screenshot:** crop the WhatsApp header, or relabel the contact name to
"City Pathology Lab" before pasting. The body text is already brand-neutral.

**To use a screenshot instead of the CSS chat:** replace the relevant `<div class="phone">…</div>`
block in the deck file (`deck-en.html` / `deck-hi.html` / `deck-mr.html`) with
`<img src="assets/page6.png" style="max-width:120mm; border-radius:6mm">` (drop the image in an
`assets/` folder next to the deck files).

---

## C. Hero photo slots (4 pages — optional, for the documentary feel)
The deck looks finished with **no photos** (each hero page is a designed dark-teal gradient with a
faint glyph). To turn a page into a full-bleed **photo background**, drop a JPG in an `assets/`
folder next to the deck files and set one attribute on that page's `<section>` (in whichever deck
you want it — `deck-en.html` / `deck-hi.html` / `deck-mr.html`):

| Page | Edit (already marked with a comment in the deck file) | Suggested image |
|---|---|---|
| **1 · Cover** | `<section class="page hero center cover" style="--hero:url('assets/cover.jpg')">` | a warm lab/reception or a hand holding a phone |
| **2 · Problem** | `<section class="page hero" style="--hero:url('assets/problem.jpg')">` | a busy front desk / ringing phone |
| **9 · Magic Moment** | `<section class="page hero" style="--hero:url('assets/night.jpg')">` | a dim lab at night / closed shutter |
| **13 · Final** | `<section class="page hero center final" style="--hero:url('assets/final.jpg')">` | a calm open lab at sunrise |

- **Size:** ~**1480 × 1050 px** landscape JPG (A4 landscape ratio). It auto-covers and centers.
- A built-in dark scrim keeps the white text legible over any photo — no editing needed.
- Use your own photos or **properly licensed** stock only (this is a commercial sales doc). Avoid a
  real, identifiable patient or the seed lab's premises.

---

## D. Honesty guardrails (keep these true in any swap)
- **Never** show the seed lab's real name, logo, or a real patient's number/report.
- The demo sends a **"report ready" text** — do **not** stage a fake PDF-in-chat. The deck already
  states the PDF-in-chat is the *production* capability (page 8).
- ROI figures stay labeled **illustrative**; pricing stays "confirmed on signup."
