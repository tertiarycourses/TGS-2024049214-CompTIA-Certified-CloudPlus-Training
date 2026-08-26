# Lab 23 — CIS Benchmark Audit with Lynis

In this lab you will audit the Killercoda VM against the **CIS Ubuntu Benchmark** using **Lynis**, score it, and remediate one finding — covering the exam's *benchmark, hardening, patching* sub-objectives.

## Lab platform

Run all commands on the **Killercoda Ubuntu Playground**:

https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — Install Lynis

```bash
apt update && apt install -y lynis
lynis --version
```

---

## Step 2 — Run a baseline audit

```bash
lynis audit system --quiet
```

When the run finishes, look for:

```
Hardening index : 60 [#######        ]
```

The **hardening index** is your CIS score. Tertiary Infotech treats anything below 80 as a fail.

---

## Step 3 — Read the warnings

```bash
grep -E '^\s+\!' /var/log/lynis.log | head -20
lynis show warnings
lynis show suggestions | head -20
```

Each suggestion maps to a CIS control number — for example *file permissions on /etc/shadow*, *unused kernel modules*, *no MOTD banner*.

---

## Step 4 — Remediate one finding (set permissions)

```bash
chmod 600 /etc/shadow /etc/gshadow
chmod 644 /etc/passwd /etc/group
ls -l /etc/shadow /etc/passwd
```

---

## Step 5 — Remediate a kernel hardening finding

> Ready-made file: [`99-hardening.conf`](99-hardening.conf) — you can download it instead of typing this block.

```bash
cat > /etc/sysctl.d/99-hardening.conf <<'EOF'
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.tcp_syncookies = 1
kernel.randomize_va_space = 2
EOF
sysctl --system | tail -10
```

---

## Step 6 — Re-audit and compare

```bash
lynis audit system --quiet
grep "Hardening index" /var/log/lynis.log
```

The score should rise.

---

## Step 7 — Vendor-specific benchmarks

Lynis covers Ubuntu/RHEL OS hardening. For cloud-vendor benchmarks use:

- **Prowler** (AWS, Azure, GCP, K8s) — https://github.com/prowler-cloud/prowler
- **kube-bench** (CIS Kubernetes) — https://github.com/aquasecurity/kube-bench
- **Docker Bench** (CIS Docker) — https://github.com/docker/docker-bench-security

Quick demo:

```bash
apt-get update
apt-get install -y wget
wget -qO- https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
```
```
cat > /tmp/docker-lab/Dockerfile <<'EOF'
FROM ubuntu:20.04

USER root

RUN apt-get update && apt-get install -y openssh-server

EXPOSE 22

CMD ["bash"]
EOF
```
```
trivy config /tmp/docker-lab
```

## Step 8 — Cleanup

```bash
rm -f /etc/sysctl.d/99-hardening.conf
sysctl --system >/dev/null
```

---

## Test it

Run these checks to prove the lab worked before you move on:

```bash
grep "Hardening index" /var/log/lynis.log
lynis show warnings
lynis show suggestions | head -20
ls -l /etc/shadow /etc/passwd
sysctl net.ipv4.tcp_syncookies kernel.randomize_va_space net.ipv4.conf.all.send_redirects
```

**Expected:** Run this before Step 8. The log shows a `Hardening index : NN [######  ]` line that is **higher after the Step 6 re-audit than the Step 2 baseline**; `/etc/shadow` is `-rw-------` (600) and `/etc/passwd` is `-rw-r--r--` (644); and the kernel now reports `net.ipv4.tcp_syncookies = 1`, `kernel.randomize_va_space = 2` and `net.ipv4.conf.all.send_redirects = 0` from the hardening sysctl file.

---

## What you learned
- Benchmarks are objective targets you can measure against.
- Hardening = closing the gap between current and benchmark.
- Different benchmarks for OS, Docker, K8s, AWS, Azure.

## Free tools used
- Lynis — https://cisofy.com/lynis
- CIS Benchmarks (free PDFs) — https://www.cisecurity.org/cis-benchmarks
- Prowler — https://github.com/prowler-cloud/prowler
- kube-bench — https://github.com/aquasecurity/kube-bench
- Docker Bench Security — https://github.com/docker/docker-bench-security

---

## Files in this lab

| File | Purpose |
|------|---------|
| [`99-hardening.conf`](99-hardening.conf) | Step 5 sysctl hardening file that raises the Lynis hardening index. |
