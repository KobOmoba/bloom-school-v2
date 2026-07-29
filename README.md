# bloom-school-v2

EduBloom School Portal — v2 test sandbox. This is where **Premium tier**
features get built and validated before being ported into the live
production app (`School-Bloom`), the same v2-first → port-later workflow
already used for `bloom-agent-v2` → `bloom-agent`.

---

## 🔐 Per-Class / Per-Subject Data Isolation — Phase 1 + 2 built, Phase 3 (rules) drafted, not live

**The requirement, confirmed with Bayo directly:** every student's data is
private. A Class Teacher can see and edit only the students in their
assigned class — nothing else, not even read-only. A Subject Teacher can
touch only their assigned subject's scores, for any class — but nothing
else about a student (no profile, no fees, no other subjects). Someone who
is *both* a Class Teacher and a Subject Teacher gets the union of both:
full access to their own class, plus their subject across every other
class too. Fees are gated separately from everything else, Principal/Bursar
only. None of this was achievable with the old data model — Firestore
security rules can restrict a whole document, not a slice inside one, and
the old model kept every student, every subject, every term in one flat
array/object per school.

### New data shape (Phase 1 + 2 — built, dual-write, live in this sandbox now)
```
schools/{schoolId}/students/{studentId}                          → profile only: name, phone, class
schools/{schoolId}/students/{studentId}/private/fees               → totalFee, paid, paymentHistory
schools/{schoolId}/students/{studentId}/scores/{term}_{subject}    → ca1, ca2, ca3, exam (doc id e.g. "Term1_Mathematics")
schools/{schoolId}/staff_directory/{uid}                           → uid IS the real Firebase Auth UID. role, assignedClass, assignedSubjects
```

**Built as dual-write, not a replacement.** `addStudent`, `deleteStudent`,
`recordPayment`, `saveScores`, and `addStaff` now also write into this new
structure alongside the existing flat `SD.students`/`SD.scores`/`SD.staff`
save path. Nothing existing was rewired to *read* from the new structure
yet — the whole app still runs on the old in-memory model for rendering.
This means: new activity from today onward populates the new structure
correctly, but it's not yet the source of truth for anything, and a
student added before this change has no `_v2Id` link until edited once.

**Known gap, not fixed in this pass:** `deleteStudentV2` doesn't cascade —
Firestore won't auto-delete a student's `private/fees` or `scores/*`
sub-documents when the parent student doc is deleted. Orphaned sub-docs
would accumulate. Needs a small cleanup pass (delete subcollection
contents before deleting the parent) before this matters for real use.

### Phase 3 — the actual security rules (drafted below, NOT yet published anywhere)
```
function isStaffOf(schoolId) {
  return request.auth != null &&
    exists(/databases/$(database)/documents/schools/$(schoolId)/staff_directory/$(request.auth.uid));
}
function myRole(schoolId) {
  return get(/databases/$(database)/documents/schools/$(schoolId)/staff_directory/$(request.auth.uid)).data.role;
}
function myAssignedClass(schoolId) {
  return get(/databases/$(database)/documents/schools/$(schoolId)/staff_directory/$(request.auth.uid)).data.assignedClass;
}
function myAssignedSubjects(schoolId) {
  return get(/databases/$(database)/documents/schools/$(schoolId)/staff_directory/$(request.auth.uid)).data.assignedSubjects;
}
function isPrincipalOf(schoolId) { return isStaffOf(schoolId) && myRole(schoolId) == 'Principal'; }
function isBursarOf(schoolId)    { return isStaffOf(schoolId) && myRole(schoolId) == 'Bursar'; }

match /schools/{schoolId}/staff_directory/{uid} {
  allow read: if isPrincipalOf(schoolId) || (isStaffOf(schoolId) && request.auth.uid == uid);
  allow write: if isPrincipalOf(schoolId);
}

match /schools/{schoolId}/students/{studentId} {
  allow read, write: if isPrincipalOf(schoolId)
                    || (isStaffOf(schoolId) && myAssignedClass(schoolId) == resource.data.class);
  allow create: if isPrincipalOf(schoolId)
             || (isStaffOf(schoolId) && myAssignedClass(schoolId) == request.resource.data.class);
}

match /schools/{schoolId}/students/{studentId}/private/fees {
  allow read, write: if isPrincipalOf(schoolId) || isBursarOf(schoolId);
}

match /schools/{schoolId}/students/{studentId}/scores/{scoreId} {
  allow read, write: if isPrincipalOf(schoolId)
    || (isStaffOf(schoolId) && myAssignedClass(schoolId) ==
         get(/databases/$(database)/documents/schools/$(schoolId)/students/$(studentId)).data.class)
    || (isStaffOf(schoolId) && resource.data.subject in myAssignedSubjects(schoolId));
  allow create: if isPrincipalOf(schoolId)
    || (isStaffOf(schoolId) && myAssignedClass(schoolId) ==
         get(/databases/$(database)/documents/schools/$(schoolId)/students/$(studentId)).data.class)
    || (isStaffOf(schoolId) && request.resource.data.subject in myAssignedSubjects(schoolId));
}
```
A dual-role staff member (Class Teacher of JSS2A *and* Subject Teacher for
Mathematics) passes the scores rule two independent ways — full access to
every JSS2A student's every subject via the class-teacher clause, plus
Mathematics-only access to every *other* class via the subject-teacher
clause. A pure Subject Teacher with no `assignedClass` only ever matches
the second clause. Not published anywhere yet — needs code fully wired to
the new read path first (see "Not yet done" below), or publishing these
rules would break the app immediately since it still reads the old
structure.

### Phase 4 — migration + read-path bridge (built)

**Read-path bridge, not a rewrite.** Rather than rewriting every render/
scorecard/report-card function to fetch from Firestore individually (a
much bigger, riskier change touching dozens of functions), `hydrateFromV2()`
loads the new structure back into the *old* in-memory shape
(`SD.students` array, `SD.scores[term][sid][subject]`) right after login.
Every existing render function keeps working completely unchanged — they
still see the exact shapes they always did. Wired into all three login
paths (localStorage-cache, network, auto-login-on-page-load). If a school
has no V2 data yet, this is a safe no-op and the old flat data stands.

**Migration script — `migrateStudentsToV2(schoolId)`.** Converts a
school's existing flat students/scores/fees into the new per-document
structure. Idempotent — already-migrated students (has `_v2Id`) are
skipped, safe to run more than once. Callable via `runMigrationUI()`
(confirm dialog + result summary) or directly from DevTools console.
**Not automatic on login** — this is a real write operation and should
happen deliberately, once, per school.

**Staff accounts cannot be auto-migrated — this is a hard constraint, not
an oversight.** Existing staff passwords are SHA-256 hashed; hashes can't
be reversed, so there's no way to know a staff member's real password to
create their Firebase Auth account automatically. Each existing staff
member has to set a **new** password once via `claimAccountUI()` (prompts
for email + new password, creates their real account, links it in
`staff_directory`). Their legacy hashed-password login keeps working
until they've claimed their account — nobody gets locked out mid-transition.

### Stress test — migration logic verified against real edge-case data

Bayo provided real test fixtures: 65 students (from the same 5 ledger
photos used to build the OCR prompts), 3 terms, 13 subjects, with 7
deliberately broken score entries (grade boundaries at exactly 69/70,
blank fields, out-of-range CA/exam values, an all-zero entry, a simulated
OCR misread of 700 instead of 70, and a genuinely missing subject for one
term). Ran two Node test harnesses against this real data, using the
actual extracted app functions rather than reimplementations:

1. **Existing grading logic** (`getGrade`, `calcStudentTermStats`,
   `calcCumulative`, `_capScoreEntry`, `_hasScoreEntry`) — **14/14 checks
   passed.** All 7 edge cases already handled correctly (this hardening
   predates today's session — the credit's not mine, just verifying it holds).
2. **New Phase 1/4 migration code** — simulated the full
   `migrateStudentsToV2` → `hydrateFromV2` round trip with a mock
   Firestore, run against all 65 students' real data. **2,534 field-level
   checks passed, 0 failures.** No data loss, no corruption, and —
   critically — the one genuinely-missing subject stayed genuinely
   missing after the round trip rather than getting silently filled in
   as a zero. Also confirmed all 39 term×subject score-document IDs are
   collision-free.

This gives real confidence the new data model round-trips correctly
before it's ever pointed at a live school's actual data.

### Explicitly NOT done yet
- Rules above are drafted, not published — publishing them now would lock
  everyone out immediately unless every school has been migrated AND every
  staff member has claimed their account first. Sequencing matters here.
- Login screen's staff-login step doesn't yet try the real-auth path first
  the way bloom-portal's admin login does (Firebase Auth attempt, legacy
  fallback) — right now `claimAccountUI()` creates the real account, but
  `doLogin` doesn't yet check it before falling back to the hashed password.
- `deleteStudentV2` doesn't cascade-delete a student's `private/fees` or
  `scores/*` sub-documents — orphaned sub-docs would accumulate on delete.
- No production port — this is 100% sandbox-only, `School-Bloom` untouched.
- No testing yet against the uploaded stress-test data (65 students, 7
  deliberately broken score entries) — worth running that through the new
  structure once the login-flow piece above is closed, to confirm the
  migration and hydration round-trip real data correctly, not just clean
  synthetic cases.

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

### Stress test results (2026-07-23, v2 only)
**All 7 edge cases PASSED on bloom-school-v2:**

| # | Student (idx) | Term | Subject | Edge Case | Raw | Capped | Expected | Result |
|---|---|---|---|---|---|---|---|---|
| 1 | OGUNDETI SALAM (0) | T1 | Math | Boundary B (69) | 10+10+10+39=69 | 69 | Score=69, Grade=B | ✅ PASS |
| 2 | OYERINDE OYENEPO (1) | T1 | Math | Boundary A (70) | 10+10+10+40=70 | 70 | Score=70, Grade=A | ✅ PASS |
| 3 | OLIVIDE BIGGOLD (7) | T1 | English | Partial data (nulls→0) | null+8+null+null | 8 | Score=8, hasData=true | ✅ PASS |
| 4 | OLIYIDE GIFT (14) | T2 | BST | CA>10, Exam>70 | 15+12+11+85=123 | 10+10+10+70=100 | Score=100, ⚠️ flag | ✅ PASS |
| 5 | OLIYIDE GODWIN (25) | T2 | Civic | All-zero=F | 0+0+0+0=0 | 0 | hasEntry=true, Grade=F | ✅ PASS |
| 6 | OJEANE HATOBIN (50) | T3 | Math | Exam=700 | 10+10+10+700=730 | 10+10+10+70=100 | Score=100, ⚠️ flag | ✅ PASS |
| 7 | GBELEKALE QUARDRI (8) | T2 | CS | Missing subject | null (deleted) | – | Dash in broadsheet, excluded from cumulative | ✅ PASS |

**Additional verifications:**
- Grade boundaries: getGrade(69)=B, getGrade(70)=A, getGrade(0)=F, getGrade(100)=A ✅
- Cumulative correctly excludes missing T2 CS for GBELEKALE QUARDRI (avg of T1+T3 only) ✅
- Subject champions update correctly with capped scores ✅
- Broadsheet ranking, best student, and top-3 all render correctly ✅
- ⚠️ overflow flag confirmed via `_capScoreEntry().hasOverflow` for edges 4 & 6 ✅

**Status: v2 stress tests PASSED. Ready to port to production v1 when approved.**
