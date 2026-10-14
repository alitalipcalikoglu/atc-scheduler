# Events

```bash
WEBHOOK_OUT_URL=http://webhook-out.internal:3009
WEBHOOK_OUT_API_KEY=<a webhook-out key with the publish role>
```

| Event | When | Data |
|---|---|---|
| `ops.scheduler.job_failed` | A run ended `failed`: a permanent error, or the retries are used up. Not for attempts that will be retried | `job`, `run`, `attempts`, `httpStatus`, `error` |

The worker publishes it, so with the split roles (`api` and `worker` processes) set `WEBHOOK_OUT_*` on the worker. Best effort: events are queued in memory and sent in the background with retries; watch `events_publish_total{result="dropped|failed"}` on the API role's `/metrics`.

Pitfalls:
- A job that is skipped because its previous run is still active is not a failure and publishes nothing.
- A flapping job publishes one event per failed run; atc-status counts repeats of the same job inside its dedupe window instead of mailing each.
