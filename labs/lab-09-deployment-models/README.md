# Lab 9 — Public, Private & Hybrid Cloud Models

In this lab you will simulate a **private** cloud (your Killercoda VM), a **public** cloud (LocalStack — a free local AWS), and connect them as a **hybrid** with a WireGuard tunnel. By the end you will see why each model trades cost, control, and elasticity differently.

## Lab platform

Run all commands on the **Killercoda Ubuntu Playground**:

https://killercoda.com/playgrounds/scenario/ubuntu

> **Ready-made files:** this lab ships [`setup.sh`](setup.sh) and [`cleanup.sh`](cleanup.sh) — run `bash setup.sh` to build everything in one go, or follow the steps below to type it yourself.

---

## Step 1 — Install tools

```bash
apt update
apt install -y docker.io wireguard wireguard-tools curl unzip

systemctl enable --now docker

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o /tmp/awscliv2.zip
unzip -q /tmp/awscliv2.zip -d /tmp
/tmp/aws/install

aws --version
docker --version
wg --version
apt update && apt install -y docker.io awscli wireguard wireguard-tools
systemctl start docker
```

---

## Step 2 — Stand up a "public" cloud with LocalStack

LocalStack is a free, open-source AWS emulator — perfect for the lab without a real account.

```bash
docker run -d \
  --name public-cloud \
  -p 4566:4566 \
  -e SERVICES=s3,ec2,iam \
  -e AWS_DEFAULT_REGION=us-east-1 \
  localstack/localstack:3.8


docker run -d --name public-cloud -p 4566:4566 \
  -e SERVICES=s3,ec2,iam \
  localstack/localstack:latest
sleep 8
curl -s http://localhost:4566/_localstack/health | head -c 200
```

Configure the AWS CLI to talk to it:

```bash
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set default.region us-east-1

aws --endpoint-url=http://localhost:4566 s3 mb s3://public-bucket


aws --endpoint-url=http://localhost:4566 s3 ls
```

This is your **public cloud** — multi-tenant API, region-based, pay-per-use.

---

## Step 3 — Stand up a "private" cloud (on-prem)

Use a local MinIO bucket — the same S3 API, but inside your data centre.

```bash
docker run -d --name private-cloud -p 9000:9000 \
  -e MINIO_ROOT_USER=admin -e MINIO_ROOT_PASSWORD=cloudplus \
  minio/minio server /data

sleep 4
docker exec private-cloud mc alias set local http://127.0.0.1:9000 admin cloudplus
docker exec private-cloud mc mb local/private-bucket
```

This is **private** / on-prem — single tenant, full control.

---

## Step 4 — Hybrid: copy data between them

```bash
echo "hybrid-payload" > /tmp/data.txt
aws --endpoint-url=http://localhost:4566 s3 cp /tmp/data.txt s3://public-bucket/
docker cp /tmp/data.txt private-cloud:/data.txt
docker exec private-cloud mc cp /data.txt local/private-bucket/

aws --endpoint-url=http://localhost:4566 s3 ls s3://public-bucket/
docker exec private-cloud mc ls local/private-bucket/
```

The same object now lives in both clouds — the basis of a **hybrid** model.

---

## Step 5 — Community cloud concept

A community cloud is shared by a group with common compliance needs (e.g. healthcare, government). You would model it as a private cloud with **multi-tenant access** restricted to the community. No infra to deploy — just policy.

---

## Step 6 — Decision matrix

| Factor | Public | Private | Hybrid | Community |
|--------|--------|---------|--------|-----------|
| CapEx | Low | High | Mixed | Shared |
| Elasticity | Best | Limited | Burst-able | Limited |
| Control | Lowest | Highest | Mixed | Shared |
| Compliance | varies | strongest | depends | strongest for sector |

---

## Step 7 — Cleanup

```bash
docker rm -f public-cloud private-cloud
```

---

## Test it

Run these checks to prove the lab worked before you move on:

```bash
curl -s http://localhost:4566/_localstack/health | head -c 200
aws --endpoint-url=http://localhost:4566 s3 ls
aws --endpoint-url=http://localhost:4566 s3 ls s3://public-bucket/
docker exec private-cloud mc ls local/private-bucket/
```

**Expected:** Run this before Step 7. LocalStack's health endpoint reports `s3` as `"available"` or `"running"`; `s3 ls` lists `public-bucket`; and **the same object `data.txt` appears in both** the LocalStack `public-bucket` listing and the MinIO `private-bucket` listing — the hybrid copy succeeded across the two clouds.

---

## What you learned
- Public, private, hybrid, and community models differ in tenancy and control.
- The **API** can stay the same (S3) across deployment models.
- Hybrid lets you burst from on-prem to public cloud.

## Free tools used
- LocalStack Community — https://www.localstack.cloud
- MinIO — https://min.io
- AWS CLI — https://aws.amazon.com/cli

---

## Files in this lab

| File | Purpose |
|------|---------|
| [`setup.sh`](setup.sh) | Runs Steps 1-4 — starts LocalStack as the public cloud, MinIO as the private cloud, and copies the same object into both. |
| [`cleanup.sh`](cleanup.sh) | Step 7 teardown — removes the `public-cloud` and `private-cloud` containers. |
