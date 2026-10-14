# Calling the other atc services

Scenario: run a nightly export of failed logins, a nightly backup, and a status re-check, all without a human.

`npm run setup` of atc-stack gives the scheduler one named key per service in `TARGET_KEYS`, each with the smallest role the call needs:

| `targetKey` | Role | Typical call |
|---|---|---|
| `atc-export` | readwrite | `POST /v1/exports` |
| `atc-queue` | write | `POST /v1/queues/:q/messages` |
| `atc-backup` | admin | `POST /v1/sources/:id/backups` |
| `atc-status` | admin | `POST /v1/monitors/:id/check` |
| `atc-render` | readwrite | `POST /v1/render` |
| `atc-payments` | write | `POST /v1/payments/:id/sync` |
| `atc-flags` | write | `PUT /v1/flags/:key/environments/:env` |
| `atc-webhook-out` | publish | `POST /v1/events` |

## Nightly export

```bash
scurl -X POST $SCHED/v1/jobs -d '{
  "name": "nightly.failed-logins",
  "schedule": { "cron": "0 2 * * *", "timezone": "Europe/Istanbul" },
  "target": { "url": "http://export.internal:3014/v1/exports", "method": "POST",
              "body": { "source": "atc-audit", "format": "csv", "filters": { "actionPrefix": "atc-auth.login", "outcome": "failure" } } },
  "targetKey": "atc-export",
  "retry": { "max": 3, "backoffSec": 60 }
}'
```

The export service answers `202` and builds the file in the background; the run counts as succeeded when the job was accepted. The file lands in atc-media as a private file (see the export examples for downloads and the optional `notifyEmail`).

## Nightly backup

```bash
scurl -X POST $SCHED/v1/jobs -d '{
  "name": "nightly.backup-audit",
  "schedule": { "cron": "30 2 * * *" },
  "target": { "url": "http://backup.internal:3017/v1/sources/atc-audit/backups", "method": "POST" },
  "targetKey": "atc-backup"
}'
```

atc-backup also runs its own interval (`BACKUP_INTERVAL_MIN`); a scheduled job is for a fixed time of day on top of it.

## Pitfalls

- Target hosts must be allowed: the stack sets `TARGET_ALLOW_HTTP`, `TARGET_ALLOW_PRIVATE` and `TARGET_ALLOWED_HOSTS` for a local or compose run; on a server list the internal host names yourself.
- A job that fails for good (permanent error or retries used up) publishes `ops.scheduler.job_failed`; see [events](events.md).
- The keys carry real power (`atc-backup` is admin: it can restore). Give jobs only the calls they need and review the job list like a firewall.
