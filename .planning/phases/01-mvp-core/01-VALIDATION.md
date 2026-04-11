---
phase: 1
slug: mvp-core
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-11
---

# Phase 1 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Vitest (unit/integration) + Playwright (E2E) + pgTAP (SQL) |
| **Config file** | `vitest.config.ts` — Wave 0 installs |
| **Quick run command** | `pnpm test:unit` |
| **Full suite command** | `pnpm test` |
| **Estimated runtime** | ~45 seconds (unit+integration), ~3 min (full with E2E) |

---

## Sampling Rate

- **After every task commit:** Run `pnpm test:unit`
- **After every plan wave:** Run `pnpm test`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 45 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 1-01-01 | 01 | 1 | AUTH-05 | integration | `pnpm test tests/rls/cross-tenant.test.ts` | ❌ W0 | ⬜ pending |
| 1-01-02 | 01 | 1 | PAYMENT-05 | unit | `pnpm test tests/db/schema.test.ts` | ❌ W0 | ⬜ pending |
| 1-02-01 | 02 | 1 | AUTH-01 | integration | `pnpm test tests/auth/otp.test.ts` | ❌ W0 | ⬜ pending |
| 1-02-02 | 02 | 1 | AUTH-04 | integration | `pnpm test tests/auth/session.test.ts` | ❌ W0 | ⬜ pending |
| 1-03-01 | 03 | 2 | ONBOARD-01 | integration | `pnpm test tests/onboarding/salon.test.ts` | ❌ W0 | ⬜ pending |
| 1-03-02 | 03 | 2 | ONBOARD-05 | E2E | `pnpm test:e2e tests/e2e/shareable-link.spec.ts` | ❌ W0 | ⬜ pending |
| 1-04-01 | 04 | 2 | BOOKING-03 | sql | `pnpm test:sql tests/sql/get_available_slots.sql` | ❌ W0 | ⬜ pending |
| 1-04-02 | 04 | 2 | BOOKING-04 | integration | `pnpm test tests/booking/concurrent-booking.test.ts` | ❌ W0 | ⬜ pending |
| 1-05-01 | 05 | 2 | BOOKING-01 | E2E | `pnpm test:e2e tests/e2e/booking-flow.spec.ts` | ❌ W0 | ⬜ pending |
| 1-05-02 | 05 | 2 | BOOKING-08 | integration | `pnpm test tests/booking/pending-payment-expire.test.ts` | ❌ W0 | ⬜ pending |
| 1-06-01 | 06 | 3 | PAYMENT-01 | integration | `pnpm test tests/payment/pawapay-webhook.test.ts` | ❌ W0 | ⬜ pending |
| 1-06-02 | 06 | 3 | PAYMENT-03 | integration | `pnpm test tests/payment/webhook-idempotent.test.ts` | ❌ W0 | ⬜ pending |
| 1-07-01 | 07 | 3 | WALKIN-01 | integration | `pnpm test tests/walkin/create.test.ts` | ❌ W0 | ⬜ pending |
| 1-07-02 | 07 | 3 | PAYMENT-02 | integration | `pnpm test tests/payment/cash.test.ts` | ❌ W0 | ⬜ pending |
| 1-08-01 | 08 | 3 | NOTIF-01 | integration | `pnpm test tests/notifications/queue.test.ts` | ❌ W0 | ⬜ pending |
| 1-08-02 | 08 | 3 | NOTIF-05 | integration | `pnpm test tests/notifications/retry.test.ts` | ❌ W0 | ⬜ pending |
| 1-09-01 | 09 | 4 | AGENDA-01 | E2E | `pnpm test:e2e tests/e2e/agenda-day-view.spec.ts` | ❌ W0 | ⬜ pending |
| 1-09-02 | 09 | 4 | AGENDA-05 | integration | `pnpm test tests/agenda/realtime.test.ts` | ❌ W0 | ⬜ pending |
| 1-10-01 | 10 | 4 | DASHBOARD-01 | integration | `pnpm test tests/dashboard/ca.test.ts` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `package.json` — add `vitest`, `@vitest/ui`, `playwright`, `@supabase/supabase-js` dev deps
- [ ] `vitest.config.ts` — configure with Supabase test env variables
- [ ] `playwright.config.ts` — configure with base URL, test dir
- [ ] `tests/setup.ts` — Supabase test client, RLS-aware helpers
- [ ] `tests/rls/cross-tenant.test.ts` — stub: two JWTs, verify zero data leak
- [ ] `tests/booking/concurrent-booking.test.ts` — stub: parallel INSERT, exactly 1 success
- [ ] `tests/payment/webhook-idempotent.test.ts` — stub: replay webhook 3× → single DB update
- [ ] `tests/sql/get_available_slots.sql` — pgTAP fixtures-based slot correctness
- [ ] `tests/booking/pending-payment-expire.test.ts` — stub: 11-min `pending_payment` → cancelled
- [ ] `.env.test` — Supabase test project credentials

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| SMS OTP reçu sur téléphone réel | AUTH-01 | Nécessite un vrai numéro Gabon | Tester avec un numéro gabonais (+241XXXXXXXX) en sandbox Twilio |
| Paiement Airtel Money end-to-end | PAYMENT-01 | Requiert sandbox pawaPay activé | Utiliser numéro test `24174345678` → vérifier webhook reçu |
| Agenda responsive mobile | AGENDA-06 | Test visuel, layout tactile | Ouvrir Chrome DevTools → iPhone 14 / Samsung Galaxy S21 |
| Lien WhatsApp partageable fonctionnel | ONBOARD-05 | Requiert WhatsApp installé | Copier le lien, ouvrir depuis WhatsApp → vérifie redirection page publique salon |
| SMS rappel 24h reçu | NOTIF-02 | Nécessite attendre ou simuler cron | Créer RDV dans 24h + déclencher cron manuellement |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 45s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
