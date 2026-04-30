---
title: Kubernetes — NFS CSI driver
description: Mount Hafiz buckets as Kubernetes PersistentVolumes via the upstream nfs-csi driver
---

# Kubernetes — NFS CSI driver

The upstream [`csi-driver-nfs`](https://github.com/kubernetes-csi/csi-driver-nfs) treats any NFSv4 server as a backing store for `PersistentVolume` objects. Pointing it at `hafiz-nfs-gateway` lets pods consume Hafiz buckets as ordinary `ReadWriteMany` volumes — no S3 SDK, no Hafiz-specific image.

## What this gets you

| Use case | Without this | With this |
|---|---|---|
| Shared scratch dir across pods | `emptyDir` per pod, no sharing | one PVC, every pod sees the same files |
| WordPress / GitLab / Nextcloud uploads | `ReadWriteOnce` block disk | `ReadWriteMany` over Hafiz |
| Postgres `pg_basebackup` archive | manual `aws s3 cp` from a sidecar | mount, write, done |
| Container registry blobs | S3 + custom config | NFS volume + standard Filesystem driver |
| Per-namespace storage class | one shared bucket per cluster | one bucket per namespace, RBAC-gated |

## Architecture

```
┌─────────────────────────────┐
│  Pod: my-app                │
│   /var/data → PVC           │
└─────────────┬───────────────┘
              │ NFSv4
              ▼
┌─────────────────────────────┐
│  csi-driver-nfs (DaemonSet) │
│   mount -t nfs4 …           │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│ hafiz-nfs-gateway           │
│   bucket=app-data           │
└─────────────┬───────────────┘
              │ S3
              ▼
┌─────────────────────────────┐
│ Hafiz cluster (3 nodes)     │
└─────────────────────────────┘
```

The CSI driver runs one Pod per node and `mount`s the share into the kubelet's mount namespace; pods see a normal directory.

## Install the CSI driver

```bash
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
  --namespace kube-system --version v4.7.0
```

That's everything on the Kubernetes side. Verify:

```bash
kubectl -n kube-system get pods -l app.kubernetes.io/name=csi-driver-nfs
# csi-nfs-controller-…   3/3   Running
# csi-nfs-node-…         3/3   Running   (one per node)
```

## Run the gateway

Pick one host on the cluster (or run it as a DaemonSet — see below) and start the gateway pointing at your Hafiz cluster:

```bash
docker run -d --name hafiz-nfs --restart=unless-stopped \
  --network host \
  -e HAFIZ_ENDPOINT=http://10.50.0.61:9000 \
  -e HAFIZ_ACCESS_KEY=hafizadmin \
  -e HAFIZ_SECRET_KEY="${HAFIZ_SECRET_KEY}" \
  -e HAFIZ_BUCKET=app-data \
  hafiz-nfs:v1.17 \
  hafiz-nfs-gateway --bind 0.0.0.0:2049
```

`--network host` is the simplest path — port 2049 is the standard NFS port, and the CSI driver expects it there. If you need a non-default port, the StorageClass `mountOptions` below carries that through.

## Define a StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: hafiz-nfs
provisioner: nfs.csi.k8s.io
parameters:
  server: 10.50.0.10              # the gateway's host
  share: /                        # one bucket per gateway
  # If using non-2049 port:
  # mountPermissions: "0775"
mountOptions:
  - vers=4.1
  - nolock
  - hard
  - proto=tcp
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

Apply: `kubectl apply -f hafiz-storageclass.yaml`.

> The `nolock` mount option is recommended until your client base has been verified against [byte-range locks](../user-guide/nfs-mount.md#whats-implemented-today) — Linux's NFS lockd setup is fiddly inside a container.

## Use it from a pod

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-uploads
spec:
  accessModes: ["ReadWriteMany"]
  storageClassName: hafiz-nfs
  resources:
    requests:
      storage: 10Gi              # ignored by NFS — sized at the bucket level
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-with-nfs
spec:
  replicas: 3
  selector: { matchLabels: { app: nginx-nfs } }
  template:
    metadata: { labels: { app: nginx-nfs } }
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          volumeMounts:
            - name: data
              mountPath: /usr/share/nginx/html
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: app-uploads
```

All three nginx pods read from / write to the same Hafiz bucket. `kubectl exec ... -- echo hi > /usr/share/nginx/html/test.html` is visible from any other pod and shows up under `aws s3 ls s3://app-data/` against the underlying Hafiz cluster.

## DaemonSet alternative

If you don't want to dedicate a host to the gateway, run it on every node and have each kubelet mount its own local instance. The StorageClass server becomes `127.0.0.1`:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata: { name: hafiz-nfs-gateway, namespace: kube-system }
spec:
  selector: { matchLabels: { app: hafiz-nfs } }
  template:
    metadata: { labels: { app: hafiz-nfs } }
    spec:
      hostNetwork: true            # ← lets the kubelet hit it on 127.0.0.1:2049
      containers:
        - name: gateway
          image: hafiz-nfs:v1.17
          args: ["hafiz-nfs-gateway", "--bind", "127.0.0.1:2049"]
          env:
            - { name: HAFIZ_ENDPOINT, value: "http://hafiz-internal:9000" }
            - { name: HAFIZ_BUCKET,   value: "app-data" }
            - { name: HAFIZ_ACCESS_KEY, valueFrom: { secretKeyRef: { name: hafiz, key: access } } }
            - { name: HAFIZ_SECRET_KEY, valueFrom: { secretKeyRef: { name: hafiz, key: secret } } }
```

That makes failure domain = single node. If a kubelet's gateway dies, only that node's pods lose mounts; recovery is one container restart.

## Per-namespace bucket isolation

Run one gateway DaemonSet per namespace, each pointing at a different bucket; bind a per-namespace StorageClass to it:

```bash
for ns in team-a team-b team-c; do
  helm install hafiz-gateway-$ns ./gateway-chart \
    --set bucket=$ns \
    --namespace $ns
done
```

RBAC on the bucket level (Hafiz IAM policies) keeps team-a from touching team-b's data even if a CSI driver bug somewhere lets a pod cross namespaces.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `MountVolume.MountDevice failed: server returned: 1` | gateway not running on `server:` host. `kubectl describe pvc` for full output. |
| Pods stuck in `ContainerCreating` for >30s | check the gateway log — kernel mount handshake not completing. |
| `Operation not permitted` on chmod | expected — S3 has no mode bits; Hafiz silently accepts the syscall. |
| `Stale file handle` after gateway restart | the in-memory state (clientids, sessions) reset. Pods will reconnect within ~lease-time (60 s). |

## Related

- [POSIX FUSE Mount](../user-guide/fuse-mount.md) — single-pod use case where the gateway feels heavy.
- [NFSv4 Gateway](../user-guide/nfs-mount.md) — protocol-level reference.
- [Cluster Peer Auth](../deployment/cluster-auth.md) — the gateway connects to the Hafiz S3 cluster via signed envelopes if `HAFIZ_CLUSTER_SHARED_SECRET` is set.
