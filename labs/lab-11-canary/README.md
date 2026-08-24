# Lab 11 — Canary Deployment with HAProxy

In this lab you will gradually shift traffic from v1 to v2 using HAProxy server weights — 90/10, then 50/50, then 100% — exactly the strategy used by AWS ALB weighted target groups, Istio canaries, and Argo Rollouts.

## Lab platform

Run all commands on the **Killercoda Ubuntu Playground**:

https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — Install HAProxy and Docker

```bash
apt update && apt install -y haproxy docker.io curl
systemctl start docker
```

---

## Step 2 — Run v1 (stable) and v2 (canary)

```bash
docker run -d --name v1 -p 8091:80 nginx:alpine
docker run -d --name v2 -p 8092:80 nginx:alpine
docker exec v1 sh -c 'echo v1 > /usr/share/nginx/html/index.html'
docker exec v2 sh -c 'echo v2 > /usr/share/nginx/html/index.html'
```

---

## Step 3 — HAProxy at 90/10 (canary 10%)

> Ready-made file: [`haproxy-canary-90-10.cfg`](haproxy-canary-90-10.cfg) — you can download it instead of typing this block.

```bash
cat > /etc/haproxy/haproxy.cfg <<'EOF'
global
    daemon
defaults
    mode http
    timeout connect 5s
    timeout client 30s
    timeout server 30s
frontend fe
    bind *:80
    default_backend app
backend app
    balance roundrobin
    server v1 127.0.0.1:8091 weight 90 check
    server v2 127.0.0.1:8092 weight 10 check
EOF
systemctl restart haproxy
```

Verify the split:

```bash
for i in $(seq 1 50); do curl -s http://localhost/; done | sort | uniq -c
```

You should see ~45 v1 / ~5 v2.

---

## Step 4 — Increase to 50/50

```bash
sed -i 's/weight 90/weight 50/; s/weight 10/weight 50/' /etc/haproxy/haproxy.cfg
systemctl reload haproxy

for i in $(seq 1 50); do curl -s http://localhost/; done | sort | uniq -c
```

---

## Step 5 — Promote v2 to 100%

> Ready-made file: [`haproxy-promote-v2.cfg`](haproxy-promote-v2.cfg) — you can download it instead of typing this block.

```bash
sed -i 's/weight 50/weight 0/; 0,/weight 0/!s/weight 0/weight 100/' /etc/haproxy/haproxy.cfg
# Simpler: rewrite cleanly
cat > /etc/haproxy/haproxy.cfg <<'EOF'
global
    daemon
defaults
    mode http
    timeout connect 5s
    timeout client 30s
    timeout server 30s
frontend fe
    bind *:80
    default_backend app
backend app
    server v1 127.0.0.1:8091 weight 0  check backup
    server v2 127.0.0.1:8092 weight 100 check
EOF
systemctl reload haproxy

for i in $(seq 1 20); do curl -s http://localhost/; done | sort | uniq -c
```

100% v2. Roll-out complete.

---



---

## Step 6 — Cleanup

```bash
docker rm -f v1 v2
systemctl stop haproxy
```

---

## Test it

Run these checks to prove the lab worked before you move on:

```bash
docker ps --filter name=v1 --filter name=v2 --format '{{.Names}}\t{{.Status}}'
systemctl is-active haproxy
grep -E 'server v[12]' /etc/haproxy/haproxy.cfg
for i in $(seq 1 20); do curl -s http://localhost/; done | sort | uniq -c
```

**Expected:** Run this before Step 7. Both `v1` and `v2` containers are **Up** and `haproxy` is `active`; the config shows the current weights for each server; and the 20-request tally matches those weights — roughly 18 `v1` / 2 `v2` at 90/10, about 10/10 at 50/50, and `20 v2` once v2 is promoted to 100%.

---

## What you learned
- Canary = small percentage of real traffic on a new version.
- Weight-based routing lets you ramp gradually and rollback instantly.
- Always pair canary with metrics/alerts.

## Free tools used
- HAProxy — https://www.haproxy.org
- Docker — https://www.docker.com

---

## Files in this lab

| File | Purpose |
|------|---------|
| [`haproxy-canary-90-10.cfg`](haproxy-canary-90-10.cfg) | Step 3 HAProxy config — 90/10 weighted split between v1 and the v2 canary. |
| [`haproxy-promote-v2.cfg`](haproxy-promote-v2.cfg) | Step 5 HAProxy config — v2 promoted to 100%, v1 kept as backup. |
