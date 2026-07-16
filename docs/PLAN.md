# PLAN — signupguard

TDD, one task per focused session. Each task lists files touched, the interfaces it produces, the test to write first, and done-criteria.

## T1 — Supabase schema (dashboard-only)

Files: `supabase/migrations/0001_init.sql`
Interfaces: tables `orgs`, `api_keys`.
Test: no vitest — verify via `supabase db push` + two throwaway users; confirm neither can read the other's `orgs`/`api_keys`, and that `select *` never surfaces `key_hash` through the API-facing view/column grants.
Done: migration applies; RLS confirmed.

## T2 — Scoring rule engine

Files: `src/score.ts`
Interfaces: `scoreSignup(input: {email, ip, cf: IncomingRequestCfProperties}, lists: {disposable: Set<string>, tor: Set<string>, vpnCidrs: CidrRange[]}): {score:number, band:'low'|'medium'|'high', reasons:string[]}`.
Test first: `test/score.test.ts` — table-driven cases for each rubric row (disposable domain, Tor IP, VPN CIDR hit, datacenter ASN, clean signal) asserting score/band/reasons.
Done: pure function, no I/O; every rubric row in the LLD has a passing test case.

## T3 — List loaders + CIDR matcher

Files: `src/lists.ts`
Interfaces: `parseDisposableDomains(raw): Set<string>`, `parseTorExitList(raw): Set<string>`, `parseVpnCidrs(raw): CidrRange[]`, `matchCidr(ip, ranges): boolean`.
Test first: `test/lists.test.ts` — fixture data per source, asserts correct parse counts and a few known-IP CIDR matches/misses (including IPv4/IPv6 edge cases).
Done: `matchCidr` is O(log n) (sorted-range binary search), tested against a list of >1000 fixture ranges for correctness, not just small cases.

## T4 — KV list refresh + cron

Files: `src/refresh.ts`, `src/worker.ts` (`scheduled` export)
Interfaces: `refreshLists(env): Promise<{disposable:number, tor:number, vpn:number, errors:string[]}>`.
Test first: `test/refresh.test.ts` — mocked source fetches, asserts KV `put` calls with correctly parsed blobs and a `list:meta` write per source.
Done: one source's fetch failure doesn't block refreshing the other two.

## T5 — API key auth + Durable Object quota

Files: `src/auth.ts`, `src/quota-do.ts`
Interfaces: `hashApiKey(key): string`, `lookupApiKey(hash, env): Promise<{orgId, plan, monthlyQuota} | null>`; DO class `QuotaCounter` with `increment(): Promise<{ok:boolean, count:number, quota:number}>` (atomic, resets on new month).
Test first: `test/auth.test.ts`, `test/quota-do.test.ts` (using `vitest-pool-workers` or a DO test harness) — asserts rejection of unknown/revoked keys, asserts the counter blocks once quota is hit and resets on period rollover.
Done: quota increment is provably atomic under concurrent test calls (no double-count).

## T6 — Handlers + router + worker entry

Files: `src/handlers.ts`, `src/router.ts`, `src/worker.ts`
Interfaces: `handleCheck`, `handleCreateKey`, `handleRevokeKey`, `handleUsage`, `handleMe`.
Test first: `test/handlers.test.ts`, `test/router.test.ts` — cover 401/429/400 paths from the LLD's API table, and that a validation-rejected request never increments quota.
Done: every failure path in the LLD's API table has a test.

## T7 — Dashboard frontend

Files: `public/index.html`, `public/keys.html`, `public/usage.html`, `public/docs.html`, `public/pricing.html`, `public/app.js`, `public/styles.css`
Test: manual QA checklist (create key, see it once, revoke it, confirm revoked key gets 401 on `/api/v1/check`).
Done: docs page shows a working curl snippet; ToS states the no-PII-retention design and that the score is advisory.

## T8 — VPN/hosting CIDR source selection (launch-blocking research task)

Files: `docs/vpn-cidr-source.md` (decision record)
Test: n/a — this is a research/decision task, not code.
Done: a specific CIDR source is chosen with its license explicitly verified and cited; only then does T4's `refreshLists` get a real (non-fixture) VPN source wired in.

## T9 — Deploy + live smoke + launch checklist

Files: `wrangler.jsonc`, `scripts/smoke.ts`
Test: `scripts/smoke.ts` runs against the deployed Worker end-to-end (create a key via the dashboard flow, call `/api/v1/check` with a known disposable-domain email, assert `band:"high"`, exhaust a small test quota) — this is the ship gate.
Done: `wrangler deploy` succeeds; smoke passes; cron confirmed populating all three KV lists (including a license-verified VPN source from T8); secrets set (`SUPABASE_SERVICE_ROLE_KEY`); pricing page + ToS live.
