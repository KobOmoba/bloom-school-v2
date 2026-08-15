## 2026-08-15 — Security: silentPull now skips sensitive keys

**Finding fixed:** Student with DevTools open could write fake scores or fees to the flat parent
Firestore document (`schools/{id}`). `silentPull()` was then reading that flat doc and overwriting
in-memory `SD.scores` / `SD.students` / `SD.attendance` — meaning hacked data would display on
the portal and print on report cards, even though the protected V2 subcollection data was untouched.

**Fix:** `silentPull()` now has a `SUBCOL_KEYS` blocklist. It only pulls `config`-level keys from the
flat parent doc. All sensitive keys (students, scores, attendance, staff, expenses, health, arts,
sports, music, alumni, socialPages, commsLog, affective) are now ignored in `silentPull()`.
V2 subcollections loaded by `hydrateFromV2()` remain the authoritative source for all of these.

**What still needs to happen before porting to School-Bloom production:**
- Test on sandbox: add a student, record scores, verify silentPull no longer overwrites them
- Confirm hydrateFromV2 loads correct data after silentPull runs

**Cache-bust:** `?v=20260815-securepull`

---
## 2026-08-14 — Security Fixes (Post-Pentest)
Pentest conducted across all three sandboxes. Three code fixes applied to ALL six repos
(production + sandbox) simultaneously.

### Fixes applied
**Finding #7 — XSS via apostrophe in student names (all three apps)**
- `esc()` now encodes single quotes: added `.replace(/'/g,"&#39;")` to prevent onclick handler breakout
- Affects: student attendance buttons, agent deal cards, portal pending list

**Finding #1 — Plaintext Principal password in publicly-readable Firestore doc (portal)**
- `confirmApproval()`: password is now SHA-256 hashed via `_sha256()` before writing to `schools/{id}.staff[0].password`
- `repairSchool()`: same hash applied
- `_sha256()` helper added to portal_app.js (uses Web Crypto API, same as school portal's own hash)
- School-Bloom `_verifyPassword()` already handles 64-char SHA-256 hashes — no school-side change needed
- Admin records in `admin_approved_schools` still store plaintext for WhatsApp sending (Bayo's private use)

**Finding #2 — Commission inflation via localStorage tampering (portal)**
- `confirmApproval()` now re-queries `admin_agents` collection to get the REAL commission rate
- Rate is capped at 30% maximum regardless of what the deal doc claims
- Fallback to deal.agent.commission (min with 30%) if agent lookup fails

### Still open (require non-code changes)
**Finding #3 — `admin_agent_requests` fully public (Firestore rules)**
- Requires manual Firestore rules update — see rules text provided separately
- Until pasted: anyone can read all applicant names/phones and spam the portal with fake requests

**Finding #4 — RBAC client-side only**
- Architecture change deferred — all data is in SD object from login, server rules protect writes
- Real fix = lazy-load fee data from subcollections only for Principal/Bursar roles

**Finding #5 — SQ flat write re-exposes students (bloom-school-v2)**
- Deferred — requires removing all SQ.push('students',...) flat writes; subcollection writes are primary
- Low risk until a Principal has claimed Firebase Auth

**Finding #6 — Fake deal submission**
- Partially mitigated: portal now ignores submitted commission rate (Fix #2)
- Full fix requires agents to use Firebase Auth for deal creation

### Cache-bust
- All six repos: `?v=20260814-security` in index.html, CACHE_NAME updated in sw.js

---
# bloom-school-v2 — School Portal PENTEST SANDBOX
**Last reset:** 2026-08-14
**Source:** School-Bloom production (school.edubloom.com.ng)

This repo is a clean copy of the current production School-Bloom codebase
used for security testing and feature development ONLY. No real school data
is entered here.

## Live sandbox URL
https://kobomoba.github.io/bloom-school-v2/

## What changed from the previous sandbox
- Wiped old V2 subcollection model code
- Replaced with exact School-Bloom production code as of 2026-08-14
- Fresh start for pentest work

## Pentest status
See PENTEST_REPORT.md (tracked separately)

## Standing rules
- Fixes proved here → ported verbatim to School-Bloom
- Cache-bust: bump ?v= in index.html + CACHE_NAME in sw.js every push
- Never commit CNAME to this repo
