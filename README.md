# signupguard

A signup-risk scoring API: POST an email + IP, get back a risk score built from disposable-email lists, Tor exit nodes, VPN/hosting CIDRs, and Cloudflare's own `cf.*` request signals.

**Status: planned — not yet built (50-SaaS challenge #24)**

## The problem

SaaS founders running free trials get hammered by disposable-email signups and VPN/Tor abuse farming multiple trials, and don't want to build and maintain their own blocklists.

## Target buyer

Indie/small-team SaaS founders fighting free-trial abuse who want a drop-in API, not a full fraud-platform contract.

## Pricing hypothesis

Rs1499/mo per 10,000 checks.

## Stack summary

Pure Cloudflare Workers + KV (no LLM, no VPS) — this is the one product in the batch with no LLM in the request path. Reference lists (disposable domains, Tor exits, VPN/hosting CIDRs) refresh nightly into KV via cron; the scoring endpoint is a synchronous KV+`cf.*` lookup. Supabase is used narrowly, for the dashboard's magic-link login and API-key/plan records only.

## How to continue this build

Nothing is implemented yet. Read `docs/LLD.md` for the architecture and data model, then `docs/PLAN.md` for the ordered TDD task list, then follow `CLAUDE.md` for how this repo relates to the shipped reference implementation.

## Risks / Constraints

- `disposable-email-domains` and public Tor exit-node lists are CC0 — safe to bundle and redistribute via KV.
- FingerprintJS **v5 is MIT; v4 is BSL and forbidden.** If client-side device fingerprinting is ever added, it must be v5 only — do not add v4.
- ipinfo.io / ipapi.co free tiers are **non-commercial** — cannot be used in a paid product. Use Cloudflare's own `cf.asn`/`cf.country`/`cf.colo` request-object signals instead.
- v1 has **no LLM** — this is deliberate, not a placeholder; pure Workers+KV keeps the scoring endpoint fast and cheap at 10k+ checks/mo per customer.
- No verified-license VPN/hosting CIDR source is picked yet — do not launch with an unverified-license list; this is a launch gate, see `docs/LLD.md`.
- The risk score is advisory, not a verdict, and the API stores emails and IPs — a privacy/retention note must ship in the docs and ToS.
