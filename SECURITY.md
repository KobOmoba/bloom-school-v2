# Security Policy — Educational Bloom

## Supported Versions

| App | Domain | Supported |
|-----|--------|-----------|
| School Portal | school.edubloom.com.ng | ✅ Active |
| Agent App | agent.edubloom.com.ng | ✅ Active |
| Command Center | portal.edubloom.com.ng | ✅ Active |

Sandbox repos (bloom-school-v2, bloom-agent-v2, bloom-portal-v2) are test environments.
Vulnerabilities found in sandboxes are valid but lower severity unless they also affect production.

## Reporting a Vulnerability

**Email:** aarinat.company.limited@gmail.com  
**WhatsApp:** +234 814 507 3941  
**Response SLA:** 48 hours acknowledgement · 7 days for critical issues

Please include:
- Description of the vulnerability
- Steps to reproduce
- Impact assessment
- Your contact details

Do NOT publish, exploit, or use the vulnerability to access data beyond your own test account.

## Security Controls

| Control | Status |
|---------|--------|
| HTTPS everywhere | ✅ Enforced |
| Firestore security rules | ✅ Active + auto-tested on every push |
| Default-deny catch-all | ✅ `allow read, write: if false` |
| Password hashing (SHA-256) | ✅ Web Crypto API |
| XSS output encoding | ✅ `esc()` encodes `<>&"'` |
| Commission validation (server) | ✅ Portal re-queries admin_agents |
| AI architecture review | ✅ Claude Sonnet on every push |
| Automated pentest | ✅ GitHub Actions (pentest-ci.js) |

## Security Pipeline

Every push to this sandbox repo triggers two automated workflows:

1. **AI Architecture Review** (`.github/workflows/ai-review.yml`)  
   Claude Sonnet reviews code changes for security and architecture issues.  
   Focus: Firestore rules, auth enforcement, data exposure, business logic attacks, NDPA compliance.  
   Posts results as a commit comment. Fails the workflow if issues are found.

2. **Firestore Rules Pentest** (`.github/workflows/pentest.yml`)  
   `pentest-ci.js` runs 15 live checks against Firestore to verify rules are correctly  
   blocking/allowing access. Runs on push and every Monday 6am Lagos time.

## NDPA 2023

Data Controller: AariNAT Company Limited (RC-1732521)  
Regulator: Nigeria Data Protection Commission (ndpc.gov.ng)  
Lawful basis: Contract + Legitimate Interest  
Retention: 24 months after subscription end  

## Changelog

| Date | Change |
|------|--------|
| 2026-08-15 | silentPull security fix — V2 subcollections are authoritative, flat doc skipped for sensitive keys |
| 2026-08-15 | Firestore catch-all changed from `if true` to `if false` |
| 2026-08-15 | admin_approved_schools locked to Bayo UID only |
| 2026-08-14 | admin_agent_requests added to Firestore rules (Finding #3) |
| 2026-08-14 | esc() fixed — apostrophe now encoded as &#39; (XSS fix) |
| 2026-08-14 | Commission rate server-validated in confirmApproval() |
| 2026-08-14 | Principal password hashed at portal write time (SHA-256) |
| 2026-08-10 | Firestore rules corrected — authed() replaced by isBayo() on admin collections |
