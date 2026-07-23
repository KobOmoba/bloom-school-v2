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

---

## 🔒 OCR Gate Bypass (Testing)

**Date:** 2026-07-23

Premium gate removed for sandbox testing. All 5 OCR scan buttons (Add Student, Add Staff, Log Expense, Record Payment, Subjects scan) are now visible and functional regardless of plan tier.

Changes made:
- `app.js`: `_isPremium()` always returns `true`
- `app.js`: `openM()` always shows scan box, hides nudge
- `app.js`: `loadSettings()` always shows subject scan box
- `app.js`: `buildProfile()` payment scan button always rendered
- `index.html`: `ns/sf/exp/subj-premium-scan` divs set to `display:block`

Restore all gates before porting to production `School-Bloom`.

---

## 📷 Camera + Gallery Picker Fix

**Date:** 2026-07-23

Removed `capture="environment"` from all 10 scan file inputs so mobile
users get the native picker offering both 📷 camera and 🖼️ gallery/files.

Inputs affected (both `index.html` and `app.js`):
- Fee register scan, CSV/photo bulk import, Subject list scan
- Student form scan, Staff ID scan, Expense receipt scan
- Payment receipt scan, Score OCR image input
- Student photo upload, Edit photo upload

---

## 🧠 OCR Prompt Strengthening

**Date:** 2026-07-23

### Payment Receipt (#4) — strengthened
Now extracts 4 extra fields beyond amount/date/method:
- `payer` — who made the payment (FROM / DEPOSITOR / REMITTER)
- `recipient` — school/account name receiving it (TO / BENEFICIARY)
- `reference` — teller number, session ID, RRR
- `account_no` — destination NUBAN (10-digit)

Payer + reference shown in feedback bar after scan. max_tokens bumped 400→500.

### Staff ID (#6) — strengthened
Now extracts 2 extra fields beyond name/email:
- `role` — job title (POSITION / DESIGNATION), auto-matched to the Role dropdown via fuzzy match
- `phone` — contact number (PHONE / TEL / MOBILE)

Role autofills dropdown when fuzzy-matched. Phone + unmatched role shown in feedback bar. max_tokens bumped 300→400.

Prompt #1 (GROQ_OCR_PROMPT) intentionally left unchanged.

---

## 📊 Report Card Score Validation (Edge Case Hardening)

**Date:** 2026-07-23

Added score validation and capping across all score rendering paths —
scorecard, report card, broadsheet, bulk entry grid, and cumulative view.
Based on the 7-edge-case stress test (`edge_case_legend.json`) covering
65 students × 13 subjects × 3 terms.

### New helpers
- `_capScoreEntry(v)` — caps CA values to 0–10, Exam to 0–70. Returns
  capped values, raw values, capped total, and `hasOverflow` flag for
  out-of-range detection (OCR misreads like exam=700).
- `_hasScoreEntry(termData, sub)` — distinguishes "student scored 0"
  from "no scores entered yet" by checking if the subject entry actually
  exists in the term data object.

### Edge cases addressed
1. **Out-of-range values** (CA>10, Exam>70) — now capped to max. ⚠️ flag
   shown on scorecard, report card, and bulk grid. Red border on
   overflowing input fields. Prevents impossible totals like 123/100.
2. **All-zero entries** — now grades F, not blank. Previously `tot>0`
   treated 0 the same as "no data". Now uses `_hasScoreEntry()` to
   distinguish genuine all-zero from missing data.
3. **OCR misread** (exam=700) — capped to 70, ⚠️ flag visible on report
   card and scorecard. Red border on the input field in bulk grid.
4. **Missing subject in one term** — `calcCumulative()` now skips
   genuine gaps (no entry = skip), doesn't treat as zero. Previously
   defaulted to all-zeros which dragged the average down.
5. **Boundary grades** (69=B, 70=A) — already correct, no change needed.

### Functions updated
- `calcStudentTermStats()` — uses `_capScoreEntry`, tracks `hasData` per subject
- `calcCumulative()` — uses `_capScoreEntry`, skips genuine gaps
- `buildScores()` / `buildTermTable()` — capped display, overflow flags, red borders
- `scorecardSetTerm()` — same treatment for term-switch rerender
- `printReportCard()` — capped values, ⚠️ flag, grade shown for all-zero
- `printAllReportCards()` — same treatment for batch print
- `printBroadsheet()` — capped cells, grade F for all-zero students
- `renderScorecard()` — `hasData` flag, grade F for all-zero
- `renderBulkScoreGrid()` — capped display, overflow flags, red borders
- `bsgUpdate()` — live cap + flag on cell change

### Test data
See `report-card-test-data-README.md` for the full test data set:
- `students_test_data.json` — 65 students, 5 classes
- `scores_clean.json` — clean baseline scores
- `scores_stress.json` — 7 injected edge cases (see `edge_case_legend.json`)
- `sample_basic4and5_mathematics_term1.csv` — single CSV for OCR import test
- `console_loader.js` — DevTools paste-to-load script
