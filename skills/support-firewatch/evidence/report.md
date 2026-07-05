# Support Firewatch — Delivery Report

## Bounty #80 — $7

## Skill Overview

The support-firewatch skill reads a support thread and SLA policy, detects sentiment, SLA breach, and churn risk signals, and emits an escalation packet only when warranted. The escalation packet routes to a human approval inbox — it pages nobody, reassigns nothing, and notifies no customer.

## Inputs

- `thread` (json, required): Array of support thread turns with customer messages and agent responses
- `sla_policy` (json, required): Object with `response_time_minutes` and optional `resolution_hours`

## Outputs

- `signals`: Object with `sentiment` (positive/neutral/negative/very_negative), `sla_breach` (boolean), `churn_risk` (none/low/medium/high). Each signal carries `_evidence` grounding it to a thread turn or policy clock.
- `escalation`: Object with `needed` (boolean), `priority` (high/medium/low/null), `context` (string). Emitted only when warranted.

## Harness Results

3/3 cases passed:
1. `sealed_escalation_flagged` — Frustrated customer, SLA breached → escalation needed, high priority
2. `refused_no_escalation_healthy_thread` — Happy customer, no breach → no escalation
3. `stop_missing_required_inputs` — Empty turns → failure/stop

## Dogfood Test

Ran the published registry skill end-to-end:
- Escalation case: `signals.sentiment=very_negative, sla_breach=true, churn_risk=medium` → `escalation.needed=true, priority=high`
- Healthy case: `signals.sentiment=positive, sla_breach=false, churn_risk=none` → `escalation.needed=false, priority=null`

Receipts verified: digest valid, content_address valid, signature valid (Ed25519).

## Published Version

- Registry: `armstrongsam25/support-firewatch@sha-78522e303a2f`
- Page: https://runx.ai/x/armstrongsam25/support-firewatch
- Hosted harness: passed 3/3

## PR

https://github.com/runxhq/runx/pull/250

## Source

https://github.com/armstrongsam25/support-firewatch (commit bd60c3a)
