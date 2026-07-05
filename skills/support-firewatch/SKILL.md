---
name: support-firewatch
description: Monitor a support queue for escalating issue clusters — multiple tickets about the same problem, increasing severity, or stale tickets — and flag them for escalation. Never sends — just detects and flags.
source:
  type: cli-tool
  command: node
  args:
    - run.mjs
runx:
  category: ops
  input_resolution:
    required:
      - queue
---

## What this skill does

Scan a snapshot of a support queue (a list of tickets) for **escalating
issue clusters** and flag each cluster for human escalation review. This
skill never sends email, SMS, chat messages, pages anyone, or modifies any
ticketing system. It produces an `runx.support.firewatch.v1` flag record per
escalating cluster that a separate governed send/notify skill can review,
approve, and deliver with its own authority grant and receipt.

A cluster is flagged for escalation when it exhibits one or more of these
escalation signals:

- **Volume spike**: multiple tickets reference the same problem signature
  (clustered by normalized subject/keyword), meeting a configurable
  `cluster_min_tickets` threshold within the queue snapshot.
- **Severity trend**: tickets in a cluster show increasing severity over time
  (e.g. `low → medium → high → urgent`), meeting `severity_trend_min_delta`.
- **Stale escalation**: one or more tickets in a cluster are older than
  `stale_after_hours` and still unresolved, indicating the issue is festering.

Each flagged cluster includes the triggering signals, the affected ticket ids,
the computed cluster signature, and a recommended escalation target — but no
outbound delivery happens here.

## When to use this skill

Use this skill when an agent has a snapshot of a support queue and needs a
safe first decision about which issue clusters are escalating and deserve
human escalation review:

- Detect clusters of tickets reporting the same problem (volume escalation).
- Detect tickets whose severity is trending upward over time.
- Detect stale unresolved tickets that are at risk of festering.
- Produce a flag record per escalating cluster for a governed notify skill.

## When not to use this skill

Do not use this skill as a message transport, pager, ticketing-system
mutator, or automatic notifier. Do not use it to send pages, post to chat
channels, change ticket status, reassign tickets, or take any live action
against a production system.

If the queue contains regulated-action tickets (refund, account deletion,
data export, billing change, password reset), the skill must not auto-flag
those for generic escalation. It should treat them as out of scope and let a
stronger authority gate handle the consequence. This skill's authority scope
is detection and flagging only.

## Procedure

1. Require `queue` to contain `tickets` (a non-empty array) and
   `snapshot_at` (ISO 8601 string).
2. Require each ticket to contain `ticket_id`, `subject`, `severity`, and
   `created_at`. Tickets missing required fields are skipped and counted as
   `invalid_ticket_count` but do not fail the run.
3. Optional `context` may include `product` and `team` to label the
   escalation recommendation.
4. Optional `escalation_policy` configures: `cluster_min_tickets` (default 3),
   `severity_trend_min_delta` (default 2), `stale_after_hours` (default 48),
   and `confidence_threshold` (default 0.6).
5. Build normalized cluster signatures from ticket subjects using keyword
   extraction (lowercased, stopwords removed, signature = sorted unique
   content keywords). Group tickets by cluster signature.
6. For each cluster with at least `cluster_min_tickets` tickets, evaluate
   escalation signals:
   a. **volume_signal**: cluster ticket count ≥ `cluster_min_tickets`.
   b. **severity_trend_signal**: the spread between the lowest and highest
      severity in the cluster (ordered low→medium→high→urgent) is ≥
      `severity_trend_min_delta`, AND severities are non-decreasing or show
      an upward jump over time.
   c. **stale_signal**: at least one ticket in the cluster is older than
      `stale_after_hours` relative to `snapshot_at` and still in an
      unresolved status.
7. A cluster is flagged when it has at least one signal AND the combined
   confidence (signal count weighted) meets `confidence_threshold`.
8. For each flagged cluster emit an `runx.support.firewatch.v1` object with
   `cluster_signature`, `signals`, `ticket_ids`, `ticket_count`,
   `max_severity`, `oldest_ticket_age_hours`, `escalation_target`, and
   `recommended_priority`.
9. If no clusters are flagged, the run still seals successfully with an empty
   `escalations` array and `escalation_count: 0` — that is a valid "nothing
   to escalate" outcome, not a failure.

## Edge cases and stop conditions

Return a stop (exit non-zero with `reason_code: process_failed`) when:

- `queue.tickets` is empty or missing — there is nothing to monitor.
- `queue.snapshot_at` is missing or not a valid ISO 8601 timestamp —
  staleness cannot be computed without a reference time.
- Every ticket in the queue is invalid (missing required fields) — the queue
  is unusable.
- The queue contains only regulated-action tickets and no general support
  tickets to monitor — the skill is out of scope for those.

Do NOT stop (seal successfully with empty escalations) when:

- The queue is valid but no clusters meet escalation thresholds — this is a
  normal "all clear" outcome.

The authority scope is detection and flagging only. The proof surface is the
sealed receipt containing the queue summary, evaluated clusters, flagged
escalations, and skipped/invalid counts. Any live notification (page, email,
chat) requires a separate `send-as` receipt.

## Output schema

```json
{
  "queue_summary": {
    "snapshot_at": "2026-07-05T10:00:00Z",
    "total_tickets": 8,
    "valid_tickets": 7,
    "invalid_ticket_count": 1,
    "cluster_count": 3,
    "escalation_count": 1
  },
  "escalations": [
    {
      "runx.support.firewatch.v1": {
        "cluster_signature": "login 500 error gateway",
        "signals": ["volume_signal", "severity_trend_signal", "stale_signal"],
        "ticket_ids": ["TKT-101", "TKT-104", "TKT-107", "TKT-110"],
        "ticket_count": 4,
        "max_severity": "urgent",
        "oldest_ticket_age_hours": 72.5,
        "escalation_target": "on-call-engineer",
        "recommended_priority": "P1"
      }
    }
  ]
}
```

## Worked example

```bash
runx skill "$PWD" \
  --input-json queue='{
    "snapshot_at": "2026-07-05T10:00:00Z",
    "tickets": [
      {"ticket_id": "TKT-101", "subject": "Login returns 500 error", "severity": "low", "status": "open", "created_at": "2026-07-02T10:00:00Z"},
      {"ticket_id": "TKT-104", "subject": "500 error on login gateway", "severity": "medium", "status": "open", "created_at": "2026-07-03T12:00:00Z"},
      {"ticket_id": "TKT-107", "subject": "Cannot login, 500 from gateway", "severity": "high", "status": "open", "created_at": "2026-07-04T09:00:00Z"},
      {"ticket_id": "TKT-110", "subject": "Login gateway 500 urgent", "severity": "urgent", "status": "open", "created_at": "2026-07-05T08:00:00Z"}
    ]
  }' \
  --input-json context='{"product": "api-gateway", "team": "platform"}' \
  --input-json escalation_policy='{
    "cluster_min_tickets": 3,
    "severity_trend_min_delta": 2,
    "stale_after_hours": 48,
    "confidence_threshold": 0.6
  }' \
  --json
```

Expected result: `queue_summary.escalation_count = 1`, the cluster is flagged
with all three signals (volume, severity trend, stale), `max_severity = urgent`,
`recommended_priority = P1`. The run does not send any message or page.

## Inputs

- `queue`: object with `snapshot_at` (ISO 8601 string) and `tickets` (array
  of ticket objects). Each ticket has `ticket_id` (string), `subject`
  (string), `severity` (one of `low`, `medium`, `high`, `urgent`),
  `status` (string, default `open`), and `created_at` (ISO 8601 string).
- `context`: optional object with `product` (string) and `team` (string) to
  label the escalation recommendation.
- `escalation_policy`: optional object with `cluster_min_tickets` (integer,
  default 3), `severity_trend_min_delta` (integer, default 2),
  `stale_after_hours` (number, default 48), and `confidence_threshold`
  (number 0–1, default 0.6).
