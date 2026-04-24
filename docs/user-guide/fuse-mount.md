# POSIX FUSE Mount

`hafiz-mount` turns a bucket into a local filesystem you can `cd` into, `cp` from, and `vim` inside. Uses the Linux FUSE kernel interface (`fuser` crate) and talks to any S3-compatible endpoint, including a Hafiz cluster.

## Install

The `hafiz-mount` binary ships with the `hafiz-fuse` crate:

```bash
cargo build --release --manifest-path crates/hafiz-fuse/Cargo.toml --bin hafiz-mount
sudo install ./crates/hafiz-fuse/target/release/hafiz-mount /usr/local/bin/
```

Your machine needs a FUSE kernel module and `fusermount3` (pre-installed on most distros).

## Mount read-only

```bash
mkdir /mnt/hafiz
hafiz-mount \
  s3://my-bucket /mnt/hafiz \
  --endpoint http://hafiz:9000 \
  --access-key AKIA... --secret-key ...
```

`ls`, `cat`, `stat`, and range reads (`dd`, `head`, `tail`) all work. Reads go straight to the backend as `GetObject` with the `Range:` header — Hafiz never pulls an entire object for a partial read.

## Mount read-write

Add `--rw`:

```bash
hafiz-mount s3://my-bucket /mnt/hafiz --rw \
  --endpoint http://hafiz:9000 \
  --access-key AKIA... --secret-key ...
```

Now:

```bash
echo "hello" > /mnt/hafiz/greet.txt         # PUT on close
cp /etc/hosts /mnt/hafiz/hosts              # PUT on close
rm /mnt/hafiz/old.log                       # DELETE
```

Writes buffer in process memory and flush to S3 as a single `PutObject` when the file is closed (`release`). Sparse writes (`seek` past EOF, then write) zero-fill correctly so the resulting object matches what you'd see on a local disk.

## Options

| Flag | Default | Description |
|---|---|---|
| `--rw` | off | Mount read-write (explicit opt-in; read-only by default). |
| `--attr-ttl SECONDS` | 30 | How long `getattr` caches are held before re-fetching from the backend. |
| `--allow-other` | off | Let non-mount users access the mount. Implies `AutoUnmount`; requires `user_allow_other` in `/etc/fuse.conf`. |
| `--region NAME` | `us-east-1` | S3 region for signing. |

Environment variables `HAFIZ_ENDPOINT`, `HAFIZ_ACCESS_KEY`, `HAFIZ_SECRET_KEY` are picked up so you don't have to repeat the credentials on the command line.

## Scope

Today's MVP covers the POSIX ops users actually hit:

- ✅ `create`, `write`, `release`, `unlink` (full round-trip)
- ✅ `lookup`, `getattr`, `readdir`, `read` (range-based)
- ❌ `mkdir`, `rmdir`, `rename` — return `EROFS`. S3 has no real directories; a proper implementation requires coordinated COPY/DELETE and is the next slice.
- ❌ Large-file multipart upload — the write buffer stays in RAM. Files bigger than a few GiB on a single mount can OOM the process; that's the other follow-up.

## Related

- [NFSv4 Gateway](nfs-mount.md) — kernel-level NFS for remote clients without installing a binary.
- [Object lifecycle](versioning.md) — writes behave the same whether the bucket has versioning on or off; versioning just means your overwrites keep history.
