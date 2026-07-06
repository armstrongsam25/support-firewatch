# Support Firewatch — Delivery Report

## Bounty #80 — $7

## Skill Overview

The support-firewatch skill reads a support thread and SLA policy, detects sentiment, SLA breach, and churn risk signals, and emits an escalation packet only when warranted. The escalation packet routes to a human approval inbox — it pages nobody, reassigns nothing, and notifies no customer.

## Inputs

- `thread` (json, required): Object with `customer` (string) and `turns` (array of `{author, text, timestamp}`)
- `sla_policy` (json, required): Object with `response_time_minutes` (number) and optional `resolution_hours` (number)

## Outputs

- `signals`: Object with `sentiment` (positive/neutral/negative/very_negative), `sla_breach` (boolean), `churn_risk` (none/low/medium/high). Each signal carries `_evidence` grounding it to a thread turn or policy clock.
- `escalation`: Object with `needed` (boolean), `priority` (high/medium/low/null), `context` (string). Emitted only when warranted.

## Harness Results

3/3 cases passed on hosted registry:
1. `sealed_escalation_flagged` — Frustrated customer, SLA breached (360m vs 60m) → escalation needed, high priority
2. `refused_no_escalation_healthy_thread` — Happy customer, no breach → no escalation
3. `stop_missing_required_inputs` — Empty turns → failure/stop

Harness receipt IDs:
- `sha256:fa1d2d553872420f744132afa28015cddeb92c0a25826a2ebdc3672cd7e97326`
- `sha256:1d93140bf634dad1bc25af8e0a9ec5e936cc9129ab98cf01965fbd1b17de039d`
- `sha256:898fac601d01841d6c7ecbebe079c667af4a931c9ff57ed3b30679151f370d03`

## Dogfood Test

Ran the published registry skill end-to-end:

- Command: `runx skill armstrongsam25/support-firewatch@sha-78522e303a2f --registry https://api.runx.ai --input-json thread='...' --input-json sla_policy='...' --json`
- Escalation case: `signals.sentiment=very_negative, sla_breach=true, churn_risk=medium` → `escalation.needed=true, priority=high`
- Receipt: `sha256:7d0450e00cad67b7b43566882ffd961490d2d50023c8d5a002b1daaf48308ed8`
- Verify verdict: valid=true, digest=valid, content_address=valid, signature=valid (production mode, kid=runx-demo-key)

Receipts verified: digest valid, content_address valid, signature valid (Ed25519, production mode).

## Published Version

- Registry: `armstrongsam25/support-firewatch@sha-78522e303a2f`
- Page: https://runx.ai/x/armstrongsam25/support-firewatch@sha-78522e303a2f
- Hosted harness: passed 3/3
- Install: `runx add armstrongsam25/support-firewatch@sha-78522e303a2f`

## PR

https://github.com/runxhq/runx/pull/250

## Source

- Repo: https://github.com/armstrongsam25/runx-support-firewatch-skill
- Commit: `1fd1ead723876ddb6f29fe648a10b69ae3a152ce`
- Source URL: https://github.com/armstrongsam25/runx-support-firewatch-skill/tree/1fd1ead723876ddb6f29fe648a10b69ae3a152ce

## runx CLI Version

`runx-cli 0.6.16` (satisfies 0.6.14 floor)

## New User Walkthrough

1. Install: `runx add armstrongsam25/support-firewatch@sha-78522e303a2f`
2. Run: `runx skill armstrongsam25/support-firewatch@sha-78522e303a2f --input-json thread='{"customer":"Acme","turns":[...]}' --input-json sla_policy='{"response_time_minutes":60}' --json`
3. Verify: `runx verify --receipt <receipt.json> --json`
