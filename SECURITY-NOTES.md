# Known Security Findings — Accepted Risk Log

## Next.js CVEs (frontend)
- **Status:** Partially remediated
- **Current version:** next@14.2.35 (patched within 14.x line)
- **Full fix requires:** next@16.x (major version, breaking change — App Router / React version implications)
- **Decision:** Deferred. 14.2.35 addresses the postcss-related and several patched CVEs within
  the 14.x branch. Remaining advisories require a major-version migration, tracked as a
  follow-up task rather than blocking Phase 1 containerization.
- **Mitigating factors:** App not yet internet-facing in this phase; will re-assess before
  production deployment (Phase 6 security hardening).
- **Tracking:** [ ] Revisit Next.js 16 migration before production go-live.

## sharp (frontend, image optimization)
- **Status:** Not remediated
- **Reason:** Fix requires breaking-change version bump (0.35.3). Confirm actual usage of
  next/image optimization in the app before prioritizing.
- **Tracking:** [ ] Confirm if `sharp`/`next/image` optimization is used; upgrade or remove if unused.

## Infrastructure Notes

### MongoDB version pin (mongo:4.4)
- **Reason:** Local dev CPU (Intel G4650) lacks AVX instruction support, required by
  MongoDB 5.0+. Image crash-loops with SIGILL on this hardware.
- **Scope:** Local development only via docker-compose.
- **Production consideration:** Cloud VMs (Phase 2+) will use modern instance types with
  AVX support — production/staging can run mongo:7 or later. Revisit compose file per
  environment, or use a managed MongoDB service (Atlas) to sidestep this entirely.
