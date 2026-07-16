# CLAUDE.md

This repo is product #24 of Febin's 50-SaaS challenge. Nothing is built yet — `docs/` is the source of truth. Read `docs/LLD.md` then `docs/PLAN.md` and execute tasks in order with TDD.

Stack conventions + the shipped reference implementation live in the private repo `febufenn-cyber/50-saas` (`contract-reviewer/`) — but this product deliberately diverges: pure Workers+KV+Durable Objects for the scoring hot path, no LLM, and Supabase scoped narrowly to dashboard auth + API-key bookkeeping.

The no-LLM decision, the CC0/BSL/non-commercial license constraints in the LLD, and the "use `cf.*` not ipinfo/ipapi" decision are verified — do not re-litigate without evidence. The VPN CIDR source is explicitly unresolved (T8) — do not pick one without verifying its license.
