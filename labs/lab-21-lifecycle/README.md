# Lab 21 — Patching & Lifecycle Management

In this lab you will simulate the lifecycle of a cloud resource: **provision → patch (minor) → upgrade (major) → test → decommission**. You will use `apt`, `unattended-upgrades`, container image tags, and clean teardown.

## Lab platform

Run all commands on the **Killercoda Ubuntu Playground**:

https://killercoda.com/playgrounds/scenario/ubuntu

> **Ready-made files:** this lab ships [`setup.sh`](setup.sh) and [`cleanup.sh`](cleanup.sh) — run `bash setup.sh` to build everything in one go, or follow the steps below to type it yourself.

---

## Step 1 — Install patching tools

```bash
apt update && apt install -y unattended-upgrades apt-listchanges docker.io
systemctl start docker
```

---

## Step 2 — Provision phase

```bash
docker run -d --name web --label lifecycle=active nginx:1.24-alpine
docker inspect -f '{{.Config.Image}}' web
```

---

## Step 3 — Apply MINOR patches (no behaviour change)

```bash
apt-get -s upgrade | head -20             # simulate
unattended-upgrade --dry-run -d 2>&1 | tail -10
```

`unattended-upgrades` automates security minors. The CV0-004 distinction:

- **Minor** = bug fixes, security patches (1.24.0 → 1.24.5)
- **Major** = breaking changes (1.24 → 1.25)

---

## Step 4 — MAJOR upgrade (test first!)

Stage the new version side-by-side:

```bash
docker run -d --name web-new nginx:1.27-alpine
docker exec web-new nginx -v
```

Run a smoke test:

```bash
curl -sI http://$(docker inspect -f '{{(index .NetworkSettings.Networks "bridge").IPAddress}}' web-new)/ | head -1
```

If it passes, swap (a Lab-10 style blue-green):

```bash
docker rm -f web
docker rename web-new web
```

---

## Step 5 — Persistent vs ephemeral data during upgrade

Persistent data (volumes, databases) **must survive** the swap. Ephemeral state (cache, /tmp) is expected to vanish.

```bash
docker volume create app-data
docker run -d --name app2 -v app-data:/var/lib/app nginx:alpine
docker exec app2 sh -c 'echo persistent > /var/lib/app/keep.txt'
docker rm -f app2
docker run --rm -v app-data:/var/lib/app alpine cat /var/lib/app/keep.txt
```

---

## Step 6 — End of support / End of life

```bash
docker pull alpine:3.10 2>&1 | head -3   # old release
docker run --rm alpine:3.10 cat /etc/alpine-release
```

Alpine 3.10 hit **end of support** in 2021. Running EoS images means **no security patches** — a cloud-governance violation in most orgs.

---

## Step 7 — Decommissioning

Tag, snapshot, then destroy.

```bash
docker commit web web-decom-$(date +%Y%m%d)
docker images | grep decom
docker rm -f web
docker volume rm app-data
docker rmi nginx:1.24-alpine 2>/dev/null
```

The **commit** is your archival snapshot; the rm/rmi is the decommission.

---

## Step 8 — Lifecycle phases summary

1. Provision (Lab 13/14 — IaC/CaC)
2. Configure (Lab 14)
3. Monitor (Lab 16-18)
4. Patch (this lab)
5. Scale (Lab 19)
6. Backup (Lab 20)
7. Decommission (this lab)

---

## Test it

Run these checks to prove the lab worked before you move on:

```bash
docker inspect -f '{{.Config.Image}}' web
docker exec web nginx -v
docker images | grep decom
docker volume ls | grep app-data

```

**Expected:** Run this before Step 7's `docker rm`/`volume rm`. The running `web` container now reports image `nginx:1.27-alpine` (the major upgrade replaced 1.24 and kept the name), `nginx -v` prints `nginx version: nginx/1.27.x`, the archival `web-decom-<date>` image is listed, the `app-data` volume still exists, and reading it prints `persistent` — the persistent data survived the container swap.

---

## What you learned
- Minor vs major upgrades have different risk profiles.
- Persistent data must be preserved across upgrades.
- EoL/EoS items must be replaced — not just patched.
- Decommissioning includes archive + revoke + delete.

## Free tools used
- apt / unattended-upgrades (built-in)
- Docker — https://www.docker.com
- Pkgs.org (find current versions) — https://pkgs.org
- endoflife.date — https://endoflife.date

---

## Files in this lab

| File | Purpose |
|------|---------|
| [`setup.sh`](setup.sh) | Runs Steps 1-5 — provisions `nginx:1.24-alpine`, simulates minor patching, performs the major upgrade swap and proves volume data survives it. |
| [`cleanup.sh`](cleanup.sh) | Step 7 decommission — commits the archival `web-decom-<date>` image, then removes the container, volume and old image. |
