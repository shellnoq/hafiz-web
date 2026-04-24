# NFSv4 Mount

Mount any Hafiz bucket over the standard NFSv4 protocol — no client software, no FUSE, no API glue. The gateway speaks Sun RPC / XDR on TCP 2049 and translates NFS operations into S3 calls against your cluster.

Works out of the box from Linux, macOS, Windows (via the built-in NFS client), and ESXi.

## Why native NFS

Most S3-compatible products either skip POSIX access entirely or push users toward FUSE. Hafiz ships a real NFSv4 gateway because:

- **Zero client install.** `mount -t nfs4 host:/ /mnt/hafiz` — any host with a kernel NFS client (every modern OS) just works.
- **Works in enterprise networks.** Unlike FUSE mounts, NFS crosses the kernel boundary the way ops teams already expect — no unprivileged helper binaries, no `/dev/fuse` permissions.
- **Compatible with existing NAS workflows.** Applications written against a home directory, a backup target, or a `/var/spool` directory don't need to learn S3 — they keep writing POSIX syscalls.

## Starting the gateway

```bash
hafiz-nfs-gateway \
  --bind 0.0.0.0:2049 \
  --endpoint http://hafiz:9000 \
  --access-key AKIA... --secret-key ... \
  --region us-east-1
```

In production you'd run this as a systemd unit or a sidecar container. The process speaks TCP; there's no UDP fallback and no `mountd` / `portmap` separate daemons to run.

## Mounting

From a Linux client:

```bash
sudo mount -t nfs4 gateway.example.com:/ /mnt/hafiz
ls /mnt/hafiz
```

From macOS:

```bash
sudo mount -t nfs -o nfsvers=4,rw gateway.example.com:/ /mnt/hafiz
```

## What's implemented today

| COMPOUND op | Status |
|---|---|
| `NULL` | full |
| `PUTROOTFH` | full |
| `GETFH` | full |
| `GETATTR` | mandatory attrs (TYPE, SIZE, CHANGE, FSID, LEASE_TIME, FILEID, etc.) |
| `READDIR`, `OPEN`, `READ`, `WRITE`, `CLOSE` | stubbed with `NFS4ERR_NOTSUPP` — coming in the next release |

You can already mount the root filehandle and inspect attributes. The read-side ops land next; writes will land with multipart support so large files don't buffer in the client.

## Security

- The gateway uses the S3 access/secret key you pass on the command line for every NFS op. Put it behind TLS and an IP ACL if you expose it beyond a trusted network.
- `NFS4ERR_MINOR_VERS_MISMATCH` is returned for NFSv4.1/4.2 clients; session semantics (the SEQUENCE op) is on the roadmap.

## Related

- [Cluster Peer Auth](../deployment/cluster-auth.md) — if your backend is a Hafiz cluster, configure `HAFIZ_CLUSTER_SHARED_SECRET` before exposing the gateway publicly.
- [FUSE Mount](fuse-mount.md) — lighter-weight POSIX access when you control the client machine.
