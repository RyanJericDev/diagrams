# Auto-Sync Schedule

## Overview

New Zendesk tickets are automatically ingested and AI-analyzed on a recurring schedule, eliminating the need for manual sync triggers. This is powered by an EventBridge scheduled rule that invokes the existing `AsyncWorkerFunction` every 15 minutes.

## How It Works

An `AutoSync` schedule event on the `AsyncWorkerFunction` fires every 15 minutes with the payload:

```json
{"task_type": "sync", "payload": {}}
```

This triggers the same `run_sync()` pipeline used by the manual `POST /api/sync/trigger` endpoint:

1. **Ingest** — Fetches new/updated tickets from Zendesk (7-day lookback window, up to 100 tickets + 20 retries per cycle)
2. **Enrich** — Runs AI analysis (classification, sentiment, summary) on each new ticket
3. **Classify** — Assigns categories and priority based on ticket content

The sync function is **idempotent** — duplicate or overlapping runs will not create duplicate data.

## Infrastructure

Defined in `template.yaml` under `AsyncWorkerFunction.Events.AutoSync`:

```yaml
AutoSync:
  Type: Schedule
  Properties:
    Schedule: rate(15 minutes)
    Input: '{"task_type": "sync", "payload": {}}'
```

## Cost

| Resource | Monthly Cost |
|---|---|
| EventBridge rule | $0 (free tier) |
| Lambda invocations (~2,880/mo) | $0 (free tier) |
| Lambda compute | ~$0.50–2.00 |
| **Total** | **~$0–2/month** |

## Monitoring

- **CloudWatch Events console** — Verify the `AutoSync` rule is firing every 15 minutes
- **CloudWatch Logs** — Check `AsyncWorkerFunction` log group for sync run completions and errors
- **Manual trigger** — `POST /api/sync/trigger` still works independently of the schedule

## Interval Rationale

15 minutes was chosen as a balance between responsiveness and cost:

- Frequent enough that new tickets are analyzed promptly
- Infrequent enough to avoid unnecessary Lambda invocations
- Each cycle is capped (100 tickets + 20 retries), so execution stays within the 15-minute Lambda timeout
