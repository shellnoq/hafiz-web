---
title: VMware vSphere — NFS datastore
description: Use Hafiz as an NFSv4.1 datastore for ESXi VM storage
---

# VMware vSphere — NFS datastore

ESXi natively understands NFS datastores: VM disks (`.vmdk`), templates, ISOs, and snapshots all live as files on a remote NFS share. Pointing a vSphere cluster at `hafiz-nfs-gateway` lets you back VMware infrastructure with Hafiz S3 storage — without any vSphere plugin, custom SDK, or licensed third-party gateway.

## Why this matters

Synology / QNAP / NetApp boxes are the default for SMB and mid-market vSphere shops because VMware natively supports NFS but has no first-class S3 datastore. Hafiz's NFSv4.1 gateway turns S3 economics + cloud durability into a real vSphere datastore option. Concrete wins:

- **Cost.** A 100 TB NetApp FAS lists at $50K+ before maintenance. Hafiz on commodity x86 + erasure coding = $5K-ish for the same usable capacity.
- **No vendor lock-in.** Hafiz is Apache-2.0; VM disks are plain `.vmdk` files in S3. Migrating off the platform = `aws s3 sync` to anywhere.
- **Snapshots cross-region.** ESXi snapshots are cheap because the underlying object store dedupes; replicate the bucket to a DR region with Hafiz cluster sync.

## Prerequisites

- vSphere 6.5+ (NFS 4.1 client first shipped in 6.5; 7.x and 8.x supported)
- A Hafiz cluster reachable from the ESXi management network
- One bucket per datastore (vSphere expects exclusive control of the share)

## Architecture

```
                    ESXi 8.0 host
                    ┌──────────────┐
                    │  vmkernel    │
                    │   NFS client │
                    └──────┬───────┘
                           │ NFSv4.1 (TCP 2049)
                           ▼
              ┌───────────────────────┐
              │ hafiz-nfs-gateway     │
              │  --bucket=vmware-ds1  │
              └────────┬──────────────┘
                       │ S3
                       ▼
              ┌───────────────────────┐
              │ Hafiz cluster         │
              │  (3 nodes, erasure)   │
              └───────────────────────┘
```

Run the gateway on a host with low-latency reach to both the ESXi cluster and the Hafiz nodes. A dedicated VM (or container on the management node) is fine; the gateway is stateless and restartable.

## Step 1 — provision the bucket

```bash
aws --endpoint-url http://hafiz:9000 s3 mb s3://vmware-ds1
```

vSphere expects to own the share, so use a dedicated bucket per datastore. Don't share the bucket with other workloads — `.vmdk` corruption silently breaks running VMs.

## Step 2 — start the gateway

```bash
docker run -d --name hafiz-vmware --restart=unless-stopped \
  --network host \
  -e HAFIZ_ENDPOINT=http://hafiz:9000 \
  -e HAFIZ_ACCESS_KEY=hafizadmin \
  -e HAFIZ_SECRET_KEY="${HAFIZ_SECRET_KEY}" \
  -e HAFIZ_BUCKET=vmware-ds1 \
  hafiz-nfs:v1.17 \
  hafiz-nfs-gateway --bind 0.0.0.0:2049
```

ESXi insists on port 2049 for NFSv4.1. `--network host` is the simplest way; alternatively bind a publicly routable IP and open the firewall.

Verify the gateway is reachable from an ESXi host:

```bash
ssh root@esxi-host
esxcli network ip connection list | grep 2049
# Or from a Linux jumpbox:
mount -t nfs4 -o vers=4.1 gateway-host:/ /mnt/probe
ls /mnt/probe                                  # should be empty
umount /mnt/probe
```

## Step 3 — add the datastore in vCenter

UI path:

1. **Inventory → Datastore → New Datastore → NFS → NFS 4.1**
2. **Server**: gateway-host IP
3. **Folder**: `/`  *(the gateway exports a single bucket as the root)*
4. **Datastore name**: `hafiz-vmware-ds1`
5. **Hosts**: select every ESXi host that should mount this share
6. Authentication: **AUTH_SYS** (Kerberos is on the [roadmap](../user-guide/nfs-mount.md))

CLI path (one host at a time):

```bash
ssh root@esxi-host
esxcli storage nfs41 add \
  --hosts=gateway-host \
  --share=/ \
  --volume-name=hafiz-vmware-ds1 \
  --auth-mode=auth_sys
```

vSphere immediately scans the share for existing VMs (none on a fresh bucket).

## Step 4 — create a VM on it

In vCenter, **New VM → Storage → hafiz-vmware-ds1**. The `.vmdk` lands in the bucket as `<vm-name>/<vm-name>-flat.vmdk` plus the descriptor `.vmdk` and the `.vmx` config file.

Smoke check from the underlying S3 view:

```bash
aws --endpoint-url http://hafiz:9000 s3 ls s3://vmware-ds1/ --recursive | head
# 2026-04-30 15:00:00     1024  test-vm/test-vm.vmx
# 2026-04-30 15:00:01      512  test-vm/test-vm.vmdk
# 2026-04-30 15:00:30 1073741824 test-vm/test-vm-flat.vmdk
```

## Production knobs

| Setting | Recommendation |
|---|---|
| **Read/write block size** | Default vSphere uses 64 KiB; the gateway honors what the kernel asks for. Bump to 1 MiB for large sequential workloads. |
| **Number of NFS connections** | ESXi opens 4–8 connections per host; gateway scales linearly with tokio worker threads. |
| **NFS heartbeat / lease** | Hafiz uses a 60 s lease (matches ESXi's expectation). Don't override unless you've measured. |
| **Path MTU** | Jumbo frames (MTU 9000) on the storage VLAN cuts CPU per byte by ~30 %. |
| **TLS / mTLS** | NFSv4 RPCSEC_GSS isn't implemented yet. Run the gateway over a private storage VLAN; don't expose it to the Internet. |

## High-availability

The gateway is stateless — restarting it doesn't lose data, only in-flight RPCs. For HA:

- **Active-passive**: keepalived + VRRP on the gateway hosts. Failover is ~5 seconds; ESXi reconnects automatically (NFSv4 grace period).
- **Active-active**: round-robin DNS works for read-heavy workloads. Avoid for write-heavy — different gateways issuing different stateids confuse the v4.1 session protocol.

## Backup + DR

Because the datastore *is* an S3 bucket, native S3 tools work:

- **Snapshots**: `aws s3 sync s3://vmware-ds1 s3://vmware-ds1-snapshots/$(date +%F)`
- **Cross-region**: Hafiz cluster sync (see [Cross-Network Replication](../deployment/cluster.md#cross-network-replication))
- **Versioning**: enable on the bucket — every overwrite of a `.vmdk` keeps the prior version, so you can roll back a corrupted VM image to the previous boot.

## Limitations

- No support for vSphere VAAI primitives (`ATOMIC TEST AND SET`, `XCOPY`). Operations like full clones fall back to host-driven copy — slower but still correct.
- pNFS not supported. Single gateway is the metadata + data path; throughput tops out at the gateway's NIC + CPU.
- VMFS-on-NFS is not a thing — stick with NFS-native datastores.
- No NFSv3 support today; ESXi 5.5 and earlier won't work. 6.5+ is fine.

## Related

- [NFSv4 Gateway](../user-guide/nfs-mount.md)
- [Cluster + Cross-Network Replication](../deployment/cluster.md)
- [Kubernetes NFS CSI](kubernetes-nfs.md) — same gateway, different consumer
