# Lab 20 — Backup & Recovery with restic

In this lab you will use **restic** — a free, open-source, encrypted, deduplicating backup tool — to perform **full**, **incremental**, and **differential**-style snapshots, store them off-site (MinIO/S3), test recoverability, and verify integrity.

## Lab platform

Run all commands on the **Killercoda Ubuntu Playground**:

https://killercoda.com/playgrounds/scenario/ubuntu

> **Ready-made files:** this lab ships [`setup.sh`](setup.sh) and [`cleanup.sh`](cleanup.sh) — run `bash setup.sh` to build everything in one go, or follow the steps below to type it yourself.

---

## Step 1 — Install restic and MinIO (off-site target)

```bash
apt update && apt install -y restic docker.io
systemctl start docker

docker run -d --name minio -p 9000:9000 \
  -e MINIO_ROOT_USER=admin -e MINIO_ROOT_PASSWORD=cloudplus \
  minio/minio server /data
sleep 4
docker exec minio mc alias set local http://127.0.0.1:9000 admin cloudplus
docker exec minio mc mb local/backups
```

---

## Step 2 — Configure restic to talk to S3-compatible storage

```bash
export RESTIC_REPOSITORY="s3:http://localhost:9000/backups"
export AWS_ACCESS_KEY_ID=admin
export AWS_SECRET_ACCESS_KEY=cloudplus
export RESTIC_PASSWORD=cloudplus

restic init
```

The repo is **encrypted at rest** with `RESTIC_PASSWORD`.

---

## Step 3 — Create source data and take a FULL backup

```bash
mkdir -p /data/app && for i in 1 2 3; do
  dd if=/dev/urandom of=/data/app/file$i bs=1M count=2 2>/dev/null
done
restic backup /data/app --tag full
restic snapshots
```

---

## Step 4 — INCREMENTAL backup (only changed blocks)

```bash
echo "new content" >> /data/app/file1
dd if=/dev/urandom of=/data/app/file4 bs=1M count=1 2>/dev/null
restic backup /data/app --tag incremental
restic stats latest
```

restic deduplicates at block level — only deltas travel.

---

## Step 5 — Differential-style: snapshot since the full

```bash
# ============================================================
# RESTIC BACKUP TO MINIO — COMPLETE LAB
# ============================================================

# Set Restic repository on MinIO
export AWS_ACCESS_KEY_ID=admin
export AWS_SECRET_ACCESS_KEY=cloudplus
export RESTIC_REPOSITORY="s3:http://127.0.0.1:9000/backups"
export RESTIC_PASSWORD="cloudplus-restic"

# Initialize Restic repository
restic init

# Create sample application data
mkdir -p /data/app

for i in 1 2 3; do
    dd if=/dev/urandom of=/data/app/file$i bs=1M count=2 2>/dev/null
done

# First full backup
restic backup /data/app --tag full

# Show snapshots
restic snapshots

# Modify existing data and add a new file
echo "new content" >> /data/app/file1

dd if=/dev/urandom of=/data/app/file4 bs=1M count=1 2>/dev/null

# Second backup
restic backup /data/app --tag incremental

# Show all snapshots
restic snapshots

# Show statistics for latest snapshot
restic stats latest

# Show files in latest snapshot
restic ls latest

# Show repository statistics
restic stats --mode restore-size
```

---

## Step 6 — Recoverability test (granular restore)

```bash
mkdir -p /restore
SNAP=$(restic snapshots --json | python3 -c 'import json,sys; print(json.load(sys.stdin)[-1]["short_id"])')
restic restore $SNAP --target /restore --include /data/app/file1
ls -lh /restore/data/app/
```

---

## Step 7 — Bulk restore (whole snapshot)

```bash
rm -rf /data/app
restic restore latest --target /
ls -lh /data/app/
```

---

## Step 8 — Integrity check

```bash
restic check --read-data-subset=10%
```

`check` re-reads a subset and verifies hashes — the **integrity** sub-objective.

---

## Step 9 — Retention policy

```bash
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 12 --prune
```

---

## Step 10 — Backup matrix

| Type | What it stores | Restore speed |
|------|---------------|---------------|
| Full | Everything | Fastest (one snapshot) |
| Incremental | Changes since last backup of any kind | Slower (chain) |
| Differential | Changes since last full | Two snapshots |

restic technically does **content-defined chunking** — every backup is a full restorable point but only sends deltas.

---

## Step 11 — Cleanup

```bash
docker rm -f minio
rm -rf /data/app /restore
```

---

## Test it

Run these checks to prove the lab worked before you move on:

```bash
restic snapshots
restic stats latest
restic check --read-data-subset=10%
ls -lh /data/app/
docker exec minio mc ls local/backups/
```

**Expected:** Run this before Step 11. `restic snapshots` lists three snapshots tagged `full`, `incremental` and `diff`; `restic stats` reports the restored size of the latest snapshot; `restic check` ends with **no errors were found**; `/data/app/` contains `file1`–`file4` again after the Step 7 bulk restore; and the MinIO `backups` bucket holds the restic repository objects (`config`, `data/`, `snapshots/`).

---

## What you learned
- Encrypted, deduplicated backups to S3-compatible storage.
- Full / incremental / differential semantics.
- Recoverability and integrity tests are mandatory — not optional.

## Free tools used
- restic — https://restic.net
- MinIO — https://min.io
- Borg (alternative) — https://www.borgbackup.org
- Duplicity (alternative) — https://duplicity.gitlab.io

---

## Files in this lab

| File | Purpose |
|------|---------|
| [`setup.sh`](setup.sh) | Runs Steps 1-3 — installs restic, starts MinIO as the off-site S3 target, initialises the encrypted repo and takes the FULL backup. |
| [`cleanup.sh`](cleanup.sh) | Step 11 teardown — removes MinIO and deletes `/data/app` and `/restore`. |
