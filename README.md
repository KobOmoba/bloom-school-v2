# bloom-school-v2

EduBloom School Portal — v2 test sandbox. This is where **Premium tier**
features get built and validated before being ported into the live
production app (`School-Bloom`), the same v2-first → port-later workflow
already used for `bloom-agent-v2` → `bloom-agent`.

---

## 📌 What This App Is

A full working copy of production `School-Bloom`, plus **OCR-powered entry
points** layered on top as the Premium-tier differentiator: instead of
typing data into forms, a Premium school can photograph the source document
and have it auto-filled. Basic-tier schools keep the exact same manual-entry
experience as production — nothing is removed, only added, and only shown
to Premium accounts.

Reads/writes the `v2_schools` Firestore collection (isolated test data —
never touches real production `schools` documents). Firebase project is
shared (`educationbloom-699ed`), same as every other EduBloom app.

---

## 🎯 Tier Model

| | Basic | Premium |
|---|---|---|
| All School-Bloom features | ✅ | ✅ |
| Add Expense — type manually | ✅ | ✅ |
| Add Expense — 📷 scan receipt | ❌ (upgrade nudge shown) | ✅ |
| Record Payment — type manually | ✅ | ✅ |
| Record Payment — 📷 scan teller/receipt | ❌ | ✅ |
| Add Student — type manually | ✅ | ✅ |
| Add Student — 📷 scan admission form/ID | ❌ | ✅ |
| Add Staff — type manually | ✅ | ✅ |
| Add Staff — 📷 scan staff ID/CV | ❌ | ✅ |
| Settings: Subjects — type/preset manually | ✅ | ✅ |
| Settings: Subjects — 📷 scan curriculum/timetable | ❌ | ✅ |
| BloomCollect, safety features | ✅ (already existed) | ✅ |

Gate is `SD.config.plan === 'premium'` — same field/mechanism BloomCollect
and staff-count limits already use in production. Not a new gate, the same
one, extended to cover OCR too.

---

## 🧠 OCR Architecture

Reuses the **existing** Fee Register Scanner infrastructure already live in
production `School-Bloom` (`_getFeeGroqKey()`, `_resizeFeeImage()`) rather
than duplicating it. One new shared function, `_callGroqGenericVision()`,
handles all four new entry points — each just supplies its own prompt.

- **Model:** `qwen/qwen3.6-27b` via direct Groq call (same model as every
  other OCR pipeline across EduBloom — confirmed the correct choice for
  free/developer tier; see `bloom-agent-v2` README for the full model
  research).
- **Reading discipline:** every prompt includes the same never-guess rule
  that fixed the ledger payment-status bug in `bloom-agent-v2` — read
  digit-by-digit, output `"UNCLEAR"` for anything not confidently legible,
  never silently guess a plausible-looking value.
- **Rate-limit handling:** `_callGroqGenericVision()` reads the
  `Retry-After` header on 429/503/529 and waits accordingly (up to 3
  retries), same lesson learned from `bloom-agent-v2`'s ledger scanner.
- **Human-in-the-loop, always:** every scan fills the form fields and stops
  — it never auto-saves. The user reviews and taps Save themselves. This
  matters most for `scanStaffID()`, which deliberately never touches the
  password field — a photo should never generate or guess a login
  credential.

### Per-feature prompts
- **Expense receipt** → vendor, description, amount, date, category (one
  of the 9 real production expense categories)
- **Payment teller/receipt** → amount, date, payment method guess
  (Bank Transfer / Cash / POS / Online)
- **Student admission form/ID** → name, parent phone, class, date of birth
- **Staff ID/CV** → name, email only (intentionally minimal — role and
  password always require human judgement)
- **Subject list/curriculum (Settings)** → bulk-extracts every distinct
  subject name from a photographed curriculum sheet or timetable, merges
  into the existing subject chip list without duplicates (case-insensitive
  match against what's already there)

---

## 📁 What Was Copied From Production

Full `School-Bloom` `app.js` (7151 lines) + `index.html` (1583 lines) +
`style.css`, with exactly one mechanical change: every
`db.collection('schools')` → `db.collection('v2_schools')` (7 occurrences),
to keep this sandbox isolated from real school data.

**Deliberately NOT copied:**
- `CNAME` — production's custom domain (`school.edubloom.com.ng`). Copying
  this would conflict with the live domain. This app lives at
  `kobomoba.github.io/bloom-school-v2/`.
- Service worker registration — **disabled** in the copied `index.html`
  (actively unregisters any existing SW instead). A cached SW fights rapid
  test iteration — the exact caching pain already fought hard in
  `bloom-agent-v2` before `?v=N` cache-busting existed. This sandbox is
  meant to be tested live against the freshest code every time.
- `manifest.json` link tag — not needed for a test app, avoids a 404.

---

## 🔜 Next Steps

1. ✅ Full production copy + collection rename
2. ✅ OCR added: expense receipt, payment teller, student admission
   form/ID, staff ID/CV, subject list/curriculum (Settings) — all
   premium-gated
3. 🔜 **Field-test all four OCR entry points** against real documents
   before considering this ready to port back into production
4. 🔜 Port working code into live `School-Bloom` once validated (mirrors
   the `bloom-agent-v2` → `bloom-agent` migration plan)
5. 🔜 CA/exam score entry OCR (from a paper mark sheet/report card) —
   discussed as a future addition, not yet built. Different document shape
   than the four above (structured columns like a ledger, not a single
   receipt/form), will likely need its own crop strategy the way the
   ledger scanner did.
6. 🔜 Onboarding walkthrough for first-time school login — separate,
   agreed idea: onboarding shouldn't end when the agent submits the deal,
   it should continue with an in-app tour showing the school how to
   navigate their new portal. Not yet scoped or built.

---

*Maintained by Claude (Anthropic). Last major update: 2026-07-19 — full
production copy + premium OCR entry points.*

