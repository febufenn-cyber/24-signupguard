# LLD — signupguard

## Architecture (request flow)

```
Customer's backend
   |
   |  POST /api/v1/check {email, ip?}   Header: X-API-Key: sg_live_...
   v
Cloudflare Worker (src/worker.ts + router.ts)
   |-- lookupApiKey(hash)              -> KV get "apikey:<hash>" {plan, org_id, ...}
   |-- checkQuota(org_id)              -> Durable Object (atomic increment, see below)
   |-- ip = ip ?? request.headers['cf-connecting-ip']
   |-- domain lookup    -> KV get "list:disposable_domains" (bloom-filter/Set blob)
   |-- tor lookup       -> KV get "list:tor_exits"
   |-- vpn/cidr lookup  -> KV get "list:vpn_cidrs" (sorted CIDR blob, binary search)
   |-- cf signals       -> request.cf.asn, request.cf.country, request.cf.colo, etc.
   |-- score = weighted combination -> {score, band, reasons[]}
   v
Response (sync, single round trip, <50ms typical)

Cron (nightly, Worker `scheduled` export):
   |-- refresh KV "list:disposable_domains" from the CC0 source dataset
   |-- refresh KV "list:tor_exits" from the official Tor exit-list
   |-- refresh KV "list:vpn_cidrs" from the chosen (license-verified) CIDR source

Dashboard (magic-link auth via supabase-js CDN, Supabase Auth only):
   |  login -> generate/revoke API keys, view usage, plan
   v
Worker (src/dashboard-handlers.ts) reads/writes Supabase `api_keys`/`orgs` tables
(this is the ONLY place Postgres is touched — the scoring hot path never hits Supabase)
```

## Data model

**No Postgres tables hold product data for v1** — the scoring engine is deliberately pure Workers+KV per the spec. Supabase provides Auth (magic link) for the human dashboard only, plus two small tables for account/billing bookkeeping:

```sql
create table public.orgs (
  id uuid primary key default gen_random_uuid(),
  owner_user_id uuid not null references auth.users(id) on delete cascade,
  plan text not null default 'trial' check (plan in ('trial','active','cancelled')),
  monthly_quota int not null default 500,   -- e.g. 500 free, 10000 on paid plan
  created_at timestamptz not null default now()
);
alter table public.orgs enable row level security;
create policy "read own org" on public.orgs for select using (auth.uid() = owner_user_id);

create table public.api_keys (
  id uuid primary key default gen_random_uuid(),
  org_id uuid not null references public.orgs(id) on delete cascade,
  key_prefix text not null,        -- e.g. "sg_live_ab12" — shown in dashboard
  key_hash text not null unique,   -- sha256 of the full key; full key shown once at creation
  revoked_at timestamptz,
  created_at timestamptz not null default now()
);
alter table public.api_keys enable row level security;
create policy "read own keys" on public.api_keys for select
  using (org_id in (select id from public.orgs where owner_user_id = auth.uid()));
-- select never returns key_hash to the client at the API layer (column allow-list).
```

**KV schema** (the actual product data):
- `apikey:<sha256(key)>` -> `{org_id, plan, monthly_quota}` — synced from Supabase on key create/revoke, read on every check for a fast path that avoids Postgres in the hot loop.
- `list:disposable_domains` -> compact Set/bloom blob, refreshed nightly.
- `list:tor_exits` -> compact Set blob, refreshed nightly.
- `list:vpn_cidrs` -> sorted CIDR range blob, refreshed nightly.
- `list:meta` -> `{source, license, last_refreshed}` per list, for the status page.

**Quota counting** uses a **Durable Object** (one instance per org, or sharded by `org_id`), not KV — KV has no atomic increment and would double-count under concurrent requests at 10k+ checks/mo. The DO holds `{count, period_start}` in memory + storage, resets monthly. This is still "Workers-native," just not literally KV, and it's the correct primitive for exact metering (same spirit as the reference's `spend_credit`/`refund_credit` atomicity, adapted to Cloudflare's own primitives instead of a Postgres RPC).

## API routes

| Route | Method | Auth | Behavior | Failure modes |
|---|---|---|---|---|
| `/api/v1/check` | POST | API key (`X-API-Key`) | Scores email+IP, returns `{score, band, reasons[]}` | invalid/revoked key (401), quota exceeded (429), malformed email/ip (400) |
| `/config.js` | GET | none | Public Supabase URL/anon key (dashboard only) | — |
| `/api/keys` | POST | Supabase JWT | Creates a new API key, syncs to KV | — |
| `/api/keys/:id/revoke` | POST | Supabase JWT (own org) | Revokes key in Supabase + KV | not found (404) |
| `/api/usage` | GET | Supabase JWT | Current-period count from the Durable Object + quota | — |
| `/api/me` | GET | Supabase JWT | Org + plan info | — |

Cron (not an HTTP route): nightly refresh of all three KV lists.

## LLM strategy

**Not applicable — v1 has no LLM in any path.** Scoring is a deterministic weighted rule set (list membership + `cf.*` signals), which is faster, cheaper, and more explainable to a paying customer than an LLM verdict would be. Revisit only if a future version adds free-text signup-form analysis.

## Scoring rubric (v1, tunable constants — not a launch gate)

A simple weighted sum, capped at 100, with each hit recorded in `reasons[]` for transparency:

| Signal | Weight | Source |
|---|---|---|
| Disposable email domain | +40 | `list:disposable_domains` (CC0) |
| Tor exit node | +35 | `list:tor_exits` (CC0) |
| Known VPN/hosting CIDR | +25 | `list:vpn_cidrs` (source TBD, license-gated) |
| Datacenter ASN via `cf.asn` | +10 | Cloudflare request object |
| Country mismatch vs. customer-configured allowlist | +10 | `cf.country` (optional, per-customer config, v2) |

`band`: `low` (<25), `medium` (25-59), `high` (>=60). Customers act on the band; raw score + reasons are for their own tuning.

## Frontend pages

`index.html` (dashboard login), `keys.html` (create/revoke API keys, shown once at creation), `usage.html` (current-period count vs quota), `docs.html` (integration snippet: `curl -H "X-API-Key: ..." -d '{"email":...,"ip":...}'`), `pricing.html`.

## Error handling + quota flow

Quota check happens via the Durable Object's atomic increment before scoring; if over quota, return 429 immediately (score is not computed, keeping the hot path cheap on rejected requests). No refund concept — this is metered API usage, not per-op credits; a failed/malformed request still doesn't count against quota (validated before the DO increment).

## Integrations and launch gates

- No OAuth, no third-party API partnerships required for the core lists (disposable-domains, Tor exits are both CC0 and self-hosted in KV).
- **Launch-blocking: VPN/hosting-provider CIDR source is not yet chosen.** Do not pin a source and launch until its license is explicitly verified (prefer CC0/permissive, e.g. community-maintained hosting-ASN lists) — a wrong assumption here is a redistribution-license risk, not just a data-quality one.
- Razorpay checkout (post-KYC, per the portfolio-wide plan) gates the paid tier; manual UPI top-up for day-1 launch.

## Security notes

- API keys stored as `key_hash` (sha256) only, never the plaintext key, in both Supabase and KV; plaintext shown once at creation.
- RLS on `orgs`/`api_keys`: owner-only access; the scoring hot path uses the service-role key from the Worker and never runs as an authenticated user.
- Input validation: reject malformed email syntax and non-IP `ip` values before touching KV or the DO (cheap 400s, no wasted quota).
- Secrets via `wrangler secret put` (`SUPABASE_SERVICE_ROLE_KEY`); no `ANTHROPIC_API_KEY` needed for v1.
- **Privacy**: the API receives and briefly touches raw emails/IPs per check. v1 does not persist the checked email/IP itself (only the aggregate count in the DO) — no `checks` log table in Postgres, specifically to minimize PII retention given India's DPDP Act and general good practice. State this plainly in the ToS.

## Out of scope for v1

Client-side device fingerprinting (FingerprintJS v5, if added later — v4 is BSL-forbidden and must never be used), per-customer configurable scoring weights, webhook/alerting integrations, IP geolocation beyond `cf.country` (no ipinfo/ipapi paid usage), a public check-history log per customer, multi-region KV replication tuning, GraphQL/batch-check endpoint.
