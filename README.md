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
