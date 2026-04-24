# Event Bridge — webhook fan-out with filters

Every S3 mutation (PUT, DELETE, CompleteMultipartUpload) can fan out to downstream systems through Hafiz's built-in event bridge. Unlike S3's `NotificationConfiguration`, the bridge runs inside the Hafiz process with its own rule-based router, retry policy, and dead-letter queue — no SNS, SQS, or EventBridge service dependency.

## Quick start: single webhook

Set one environment variable:

```bash
HAFIZ_EVENT_WEBHOOK_URL=https://hooks.example.com/hafiz
```

That automatically registers a **catch-all** rule + target. Every mutating call now POSTs a JSON payload:

```json
{
  "event_name": "s3:ObjectCreated:Put",
  "bucket": "my-bucket",
  "key": "uploads/photo.jpg",
  "size": 284392,
  "content_type": "image/jpeg",
  "metadata": { "user-id": "42" },
  "time": "2026-04-24T15:17:54.891Z"
}
```

Optional header for bearer auth / HMAC signatures from your proxy layer:

```bash
HAFIZ_EVENT_WEBHOOK_AUTH_HEADER="Bearer $(cat /run/secrets/webhook_token)"
```

## Event types

| Event name | When |
|---|---|
| `s3:ObjectCreated:Put` | plain `PutObject` |
| `s3:ObjectCreated:CompleteMultipartUpload` | multipart finalization |
| `s3:ObjectRemoved:Delete` | `DeleteObject`, versioned delete, and delete-marker creation |

## Rules and filters

Behind the scenes the webhook URL registers one rule + one target. In cluster mode you can register multiple rules via the admin API to route different buckets / prefixes to different targets:

```json
{
  "id": "ml-pipeline",
  "bucket": "raw-datasets",
  "filter": {
    "rules": [
      { "op": "prefix", "value": "2026/" },
      { "op": "suffix", "value": ".parquet" },
      { "op": "size_gt", "value": "1048576" }
    ]
  },
  "target_ids": ["ml-trigger"]
}
```

Supported filter operators: `prefix`, `suffix`, `size_gt`, `size_lt`, `regex` (on the key), `metadata_equals`.

## Delivery guarantees

- **Async.** `dispatch()` returns immediately; actual HTTP POSTs run in spawned tokio tasks.
- **Retry with exponential back-off.** 3 attempts by default, 100 ms → 200 ms → 400 ms (capped at 5 s).
- **Dead-letter queue.** Failed events are kept in-memory (capped at 1000 per target) so an operator can inspect them through the admin API.
- **Per-target HTTP client.** Default 10 s timeout; configurable at target registration.

## Observability

At startup the server logs whether the bridge has any rules:

```
EventBridge: webhook target registered (default-webhook)
```

At dispatch time with `HAFIZ_LOG_LEVEL=debug`:

```
EventBridge: event s3:ObjectCreated:Put matched 1 rule(s)
```

## Related

- [Real-Time Change Stream](../architecture/components.md) — lower-level SSE stream for clients that want to subscribe directly instead of polling a webhook.
- [Object transform pipeline](../architecture/components.md#transform-pipeline) — same event trigger mechanism used for async thumbnail/EXIF-strip jobs.
