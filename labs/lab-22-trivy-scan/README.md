# Lab 22 — Vulnerability Scanning with Trivy

In this lab you will run **Trivy** against a container image, the local filesystem, and an IaC config. You will follow the exam's vulnerability-management steps: **scope → identify → assess → remediate**.

## Lab platform

Run all commands on the **Killercoda Ubuntu Playground**:

https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — Install Trivy

```bash
apt update && apt install -y wget docker.io
systemctl start docker
wget -qO- https://aquasecurity.github.io/trivy-repo/deb/public.key | apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb generic main" > /etc/apt/sources.list.d/trivy.list
apt update && apt install -y trivy
trivy --version
```

---

## Step 2 — Scoping

Pick what to scan. For this lab:
1. A known-vulnerable image (`vulnerables/web-dvwa`)
2. The Killercoda VM filesystem
3. A Terraform file from Lab 13

---

## Step 3 — Identify (image scan)

```bash
trivy image --severity HIGH,CRITICAL alpine:3.10 | head -50
```

Each finding includes a **CVE** ID, package, fixed version, and CVSS score.

---

## Step 4 — Assess

CVSS = severity × exploitability. The exam expects you to know CVE/CVSS terms.

```bash
trivy image --severity CRITICAL --format json alpine:3.10 | head -c 600
```

Group by severity:

```bash
trivy image alpine:3.10 --format table | tail -5
```

---

## Step 5 — Remediate (rebuild with patched base)

```bash
trivy image alpine:3.20 --severity CRITICAL | tail -10
```

`alpine:3.20` ships with patched packages. Remediation is usually:
1. Bump the base image.
2. Rebuild the application image.
3. Push and redeploy.

---

## Step 6 — Filesystem scan

```bash
trivy fs --security-checks vuln,secret /etc | head -30
```

`secret` detection finds API keys / private keys committed by mistake.

---

## Step 7 — IaC scan (misconfiguration)

> Ready-made file: [`main.tf`](main.tf) — you can download it instead of typing this block.

```bash
cat > /tmp/tf-bad/main.tf <<'EOF'
resource "aws_s3_bucket" "bad" {
  bucket = "world-readable"
}

resource "aws_s3_bucket_acl" "bad" {
  bucket = aws_s3_bucket.bad.id
  acl    = "public-read"
}
EOF

trivy config /tmp/tf-bad
```

Trivy flags the public-read S3 bucket — an **outdated/insecure component definition**.

---

## Step 8 — Daily automation hint

Add to a CI pipeline (Lab 30):

```bash
trivy image --exit-code 1 --severity CRITICAL myorg/myapp:latest
```

Non-zero exit fails the build — this is the "gate" that stops vulnerable images from shipping.

---

## Test it

Run these checks to prove the lab worked before you move on:

```bash
trivy --version
trivy image --severity HIGH,CRITICAL alpine:3.10 | tail -20
trivy image --severity CRITICAL alpine:3.20 | tail -10
trivy config /tmp/tf-bad
trivy fs --security-checks vuln,secret /etc | head -20
```

**Expected:** Trivy prints its version and a vulnerability DB date; the `alpine:3.10` scan lists **multiple HIGH/CRITICAL CVEs** with CVE IDs, installed and fixed versions; the same scan of `alpine:3.20` reports **no CRITICAL vulnerabilities** (`Total: 0`), showing that bumping the base image is the remediation; and `trivy config` flags the `/tmp/tf-bad/main.tf` public-read S3 bucket as a HIGH misconfiguration.

---

## What you learned
- Vulnerability scanning closes the **identify** step.
- CVE + CVSS is the universal vocabulary.
- Image, filesystem, and IaC scans are all part of cloud security hygiene.

## Free tools used
- Trivy — https://aquasecurity.github.io/trivy
- Grype (alternative) — https://github.com/anchore/grype
- OpenVAS / Greenbone CE — https://www.greenbone.net
- NVD CVE database — https://nvd.nist.gov
- CVE.org — https://www.cve.org

---

## Files in this lab

| File | Purpose |
|------|---------|
| [`main.tf`](main.tf) | Step 7 deliberately insecure Terraform — the public-read S3 bucket Trivy flags. |
