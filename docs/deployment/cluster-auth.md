# Cluster Peer Authentication

When you run more than one Hafiz node, the peers talk to each other over HTTP on `/cluster/join`, `/cluster/heartbeat`, and `/cluster/message`. Those endpoints bypass SigV4 on purpose — SigV4 is a client-side protocol and it makes no sense for internal gossip. That leaves an authentication gap that Hafiz closes with a dedicated HMAC layer.

## What's protected

Every peer request is wrapped in a **SignedEnvelope**:

```json
{
  "v": 1,
  "ts": 1713000000,
  "nonce": "<16-byte base64 random>",
  "payload": "<serialized ClusterMessage>",
  "sig": "<HMAC-SHA256 over v1\\n{ts}\\n{nonce}\\n{payload}, base64>"
}
```

Receivers verify:

- **Signature.** Re-computed with the shared secret; compared in constant time.
- **Freshness.** ±300 seconds against the receiver's clock. Replays past that window are rejected.
- **Version.** Only `v=1` is accepted today — bumping the version lets us change the MAC input format without silently re-accepting old senders.

Wrong-secret, tampered payloads, modified timestamps, and stale envelopes all produce `HTTP 401`.

## Rollout

Set `HAFIZ_CLUSTER_SHARED_SECRET` to the **same** value on every node. Generate one with:

```bash
openssl rand -hex 32
```

Using `docker-compose.cluster.yml` the variable is `required` — compose refuses to start without it, so you can't accidentally deploy an unauthenticated cluster.

```bash
echo "HAFIZ_CLUSTER_SHARED_SECRET=$(openssl rand -hex 32)" >> .env
docker compose -f docker-compose.cluster.yml up -d
```

## Behavior when unset (legacy mode)

If you leave `HAFIZ_CLUSTER_SHARED_SECRET` empty, Hafiz still boots, but the startup logs print a loud warning:

```
WARN cluster peer auth DISABLED — HAFIZ_CLUSTER_SHARED_SECRET is unset.
     Any network peer knowing the cluster name can inject messages.
```

That mode is kept purely for staged migrations — set the secret on every node, restart in a rolling order, done. Don't ship production without it.

## Observability

On boot with the secret set:

```
INFO cluster peer auth ENABLED — requests to /cluster/* will be HMAC-signed
```

When a stale-version node or an attacker probes the cluster endpoints:

```
WARN cluster: unsigned payload rejected (secret configured) — body[0..120]=…
WARN cluster: envelope rejected: cluster peer auth: signature mismatch
WARN cluster: envelope rejected: cluster peer auth: envelope timestamp skew too large (600s)
```

The first 120 chars of the rejected body are logged so an operator can tell whether the source is a legit misconfigured peer or an attacker probe.

## Rotating the secret

1. Update `HAFIZ_CLUSTER_SHARED_SECRET` on every node.
2. Rolling-restart the nodes **in the same order** they were first started (seed last).
3. During the rolling restart, half the cluster temporarily speaks the new secret and half the old. Heartbeats fail on the mismatched pair until the last node restarts — this is expected and converges automatically.

## Threat model

**What this protects against:**

- Rogue node on the same L2/L3 network injecting forged heartbeats or replication events.
- Someone who scraped `HAFIZ_CLUSTER_NAME` from logs or config files.
- Replay of a captured heartbeat past 5 minutes.

**What this does NOT replace:**

- Transport encryption — pair with `cluster_tls_enabled = true` for wire confidentiality.
- Admin-plane auth — the `/api/v1/cluster/*` admin endpoints still require SigV4.
- Node-identity auth — two nodes that both hold the secret are both trusted; use mTLS if you need per-node identity.

## Related

- [Security Architecture](../architecture/security.md) — where peer auth fits in the threat model.
- [TLS Configuration](tls.md) — transport-level encryption for the peer channel.
