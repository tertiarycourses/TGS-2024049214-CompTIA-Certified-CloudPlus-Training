# Lab 27 — Web Application Firewall with ModSecurity

In this lab you will deploy **Nginx + ModSecurity + the OWASP Core Rule Set (CRS)** and watch it block SQL injection, XSS, and path-traversal attempts.

## Lab platform

Run all commands on the **Killercoda Ubuntu Playground**:

https://killercoda.com/playgrounds/scenario/ubuntu

> **Ready-made files:** this lab ships [`setup.sh`](setup.sh), [`cleanup.sh`](cleanup.sh) and [`exclusions.conf`](exclusions.conf) — run `bash setup.sh` to build everything in one go, or follow the steps below to type it yourself.

---

## Step 1 — Run a pre-built Nginx + ModSecurity image

```bash
apt update && apt install -y docker.io curl
systemctl enable --now docker

docker rm -f waf backend 2>/dev/null || true
docker network rm wafnet 2>/dev/null || true

docker network create wafnet

docker run -d \
  --name backend \
  --network wafnet \
  nginx:alpine

docker run -d \
  --name waf \
  --network wafnet \
  -p 80:8080 \
  -e BACKEND=http://backend:80 \
  -e BLOCKING_PARANOIA=1 \
  -e DETECTION_PARANOIA=1 \
  -e MODSEC_RULE_ENGINE=on \
  -e MODSEC_AUDIT_ENGINE=RelevantOnly \
  owasp/modsecurity-crs:nginx

sleep 10

docker ps --filter name=waf --filter name=backend
curl -s -o /dev/null -w "health=%{http_code}\n" http://localhost/healthz
```

This image bundles Nginx + libmodsecurity + OWASP CRS — the same WAF logic AWS WAF and Cloudflare WAF base their managed rules on.

---

## Step 2 — Run a benign request (allowed)

```bash
curl -s -o /dev/null -w "benign=%{http_code}\n" \
  "http://localhost/?q=cats"
```

---

## Step 3 — Trigger a SQL injection rule

```bash
curl -s -o /dev/null -w "sqli=%{http_code}\n" \
  --get \
  --data-urlencode "id=1' OR '1'='1" \
  http://localhost/
```

You should see **403** — CRS rule 942100 (SQLi).

---

## Step 4 — Trigger an XSS rule

```bash
curl -s -o /dev/null -w "xss=%{http_code}\n" \
  --get \
  --data-urlencode 'msg=<script>alert(1)</script>' \
  http://localhost/
```

403 — CRS rule 941100 (XSS).

---

## Step 5 — Trigger a path-traversal / LFI rule

```bash
curl -s -o /dev/null -w "lfi=%{http_code}\n" \
  --get \
  --data-urlencode 'file=../../../../etc/passwd' \
  http://localhost/
```

403 — CRS rule 930100.

---

## Step 6 — Inspect the WAF audit log

```bash
docker rm -f waf
touch /tmp/modsec_audit.log

docker run -d \
  --name waf \
  --network wafnet \
  -p 80:8080 \
  -e BACKEND=http://backend:80 \
  -e BLOCKING_PARANOIA=1 \
  -e DETECTION_PARANOIA=1 \
  -e MODSEC_RULE_ENGINE=on \
  -e MODSEC_AUDIT_ENGINE=RelevantOnly \
  -e MODSEC_AUDIT_LOG=/var/log/modsec_audit.log \
  -v /tmp/modsec_audit.log:/var/log/modsec_audit.log \
  owasp/modsecurity-crs:nginx

sleep 10

curl -s -o /dev/null \
  -w "sqli=%{http_code}\n" \
  --get \
  --data-urlencode "id=1' OR '1'='1" \
  http://localhost/

curl -s -o /dev/null \
  -w "xss=%{http_code}\n" \
  --get \
  --data-urlencode 'msg=<script>alert(1)</script>' \
  http://localhost/

curl -s -o /dev/null \
  -w "lfi=%{http_code}\n" \
  --get \
  --data-urlencode 'file=../../../../etc/passwd' \
  http://localhost/

tail -40 /tmp/modsec_audit.log
```

You will see the matched rule IDs, the request, and the score.

---

## Step 7 — Tune (false-positive triage)

ModSecurity uses an **anomaly score**. The default block threshold is 5. To accept a known-good pattern:

```bash
docker exec waf sh -c \
  'find /etc/modsecurity.d -type f | sort | grep -Ei "exclusion|custom"'
```

In production you tune via test → staging → prod, never prod-first.

---

## Step 8 — Network ACL vs WAF vs Security Group

| Layer | Inspects | Tool used |
|-------|---------|-----------|
| Network ACL (Lab 3) | IP/port | iptables |
| Security Group (Lab 3) | IP/port stateful | iptables -m conntrack |
| WAF (this lab) | HTTP semantics | ModSecurity / CRS |

A WAF sees what a firewall cannot — request bodies, query strings, HTTP headers.

---

## Step 9 — Cleanup

```bash
docker rm -f waf
```

---

## Test it

Run these checks to prove the lab worked before you move on:

```bash
docker ps -a --filter name=waf
docker ps -a --filter name=backend
docker network ls | grep wafnet
```

**Expected:** Run this before Step 9. The `waf` container is **Up**; the benign request returns `benign=200` while all three attacks return **403** (`sqli=403`, `xss=403`, `lfi=403`); and the audit log tail shows the matching OWASP CRS rule IDs — 942100 for SQLi, 941100 for XSS and 930100 for path traversal — together with the anomaly score that crossed the threshold of 5.

---

## What you learned
- WAFs operate on HTTP semantics, not just IP/port.
- OWASP CRS catches the OWASP Top 10 out of the box.
- Anomaly scoring + tuning is the production workflow.

## Free tools used
- ModSecurity — https://github.com/owasp-modsecurity/ModSecurity
- OWASP Core Rule Set — https://coreruleset.org
- Cloudflare WAF (free tier) — https://www.cloudflare.com/application-services/products/waf

---

## Files in this lab

| File | Purpose |
|------|---------|
| [`setup.sh`](setup.sh) | Runs Step 1 — starts the Nginx + ModSecurity + OWASP CRS container at paranoia level 1 (the Step 2-5 attack probes stay yours to fire). |
| [`cleanup.sh`](cleanup.sh) | Step 9 teardown — removes the `waf` container. |
| [`exclusions.conf`](exclusions.conf) | Step 7 false-positive tuning rule — removes CRS rule 941100. |
