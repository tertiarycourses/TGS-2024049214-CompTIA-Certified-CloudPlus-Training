# Lab 13 — Infrastructure as Code with Terraform

In this lab you will write a Terraform configuration that provisions an S3 bucket and a security policy on **LocalStack** (free local AWS). You will see plan/apply, state, drift detection, and versioning.

## Lab platform

Run all commands on the **Killercoda Ubuntu Playground**:

https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — Install Terraform and LocalStack

```bash
apt update && apt install -y wget unzip docker.io
systemctl start docker

wget -q https://releases.hashicorp.com/terraform/1.9.5/terraform_1.9.5_linux_amd64.zip
unzip -o terraform_1.9.5_linux_amd64.zip -d /usr/local/bin
terraform version

docker run -d --name localstack -p 4566:4566 -e SERVICES=s3,iam localstack/localstack:latest
sleep 8
```

---

## Step 2 — Write the Terraform configuration

> Ready-made file: [`main.tf`](main.tf) — you can download it instead of typing this block.

```bash
# ============================================================
# COMPLETE FIX — LOCALSTACK + AWS CLI + TERRAFORM
# ============================================================

# 1. Make sure Docker is running
systemctl enable --now docker

# 2. Make sure LocalStack 3.8 is running
docker rm -f public-cloud 2>/dev/null || true

docker run -d \
  --name public-cloud \
  -p 4566:4566 \
  -e SERVICES=s3,ec2,iam \
  -e AWS_DEFAULT_REGION=us-east-1 \
  localstack/localstack:3.8

sleep 10

# 3. Verify LocalStack
docker ps --filter name=public-cloud

curl -s http://localhost:4566/_localstack/health

# 4. Install AWS CLI v2
cd /tmp

rm -rf aws awscliv2.zip

wget -q "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" \
  -O awscliv2.zip

unzip -q awscliv2.zip

./aws/install --update

# 5. Verify AWS CLI
aws --version

# 6. Configure AWS CLI for LocalStack
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set default.region us-east-1

# 7. Test LocalStack S3
aws --endpoint-url=http://localhost:4566 s3 ls

# 8. Create Terraform working directory
mkdir -p /tmp/tf
cd /tmp/tf

# 9. Create Terraform configuration
cat > main.tf <<'EOF'
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region                      = "us-east-1"
  access_key                  = "test"
  secret_key                  = "test"
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true
  skip_region_validation      = true

  endpoints {
    s3  = "http://localhost:4566"
    iam = "http://localhost:4566"
    ec2 = "http://localhost:4566"
  }

  s3_use_path_style = true
}

variable "env" {
  default = "dev"
}

resource "aws_s3_bucket" "data" {
  bucket = "tf-${var.env}-data"
}
EOF

# 10. Format and validate Terraform
terraform fmt
terraform validate

# 11. Initialize Terraform
terraform init

# 12. Create execution plan
terraform plan

# 13. Apply Terraform
terraform apply -auto-approve

# 14. Verify bucket using AWS CLI
aws --endpoint-url=http://localhost:4566 s3 ls
```

---

## Step 3 — Init, plan, apply


```bash
terraform state list
terraform state show aws_s3_bucket.data
ls -lh terraform.tfstate
```

The state file is the source of truth — back it up to a remote backend in production.

---

## Step 4 — Drift detection

Manually mutate the bucket outside Terraform:

```bash
docker exec localstack awslocal s3api put-bucket-tagging \
  --bucket cloudplus-dev-data \
  --tagging 'TagSet=[{Key=DriftedBy,Value=human}]'

terraform plan
```

`terraform plan` reports the **drift** — Terraform wants to remove the manual tag.

---

## Step 5 — Version your code (Git)

```bash
apt install -y git
cd /tmp/tf
git init -q
echo 'terraform.tfstate*' > .gitignore
echo '.terraform/' >> .gitignore
git add . && git -c user.email=lab@x -c user.name=lab commit -q -m "initial infra"
git log --oneline
```

Code reviewable, diff-able, rollback-able.

---

## Step 6 — Apply a change (test → apply pattern)

```bash
sed -i 's/env" { default = "dev"/env" { default = "prod"/' main.tf
terraform plan
terraform apply -auto-approve
terraform state list
```

Terraform creates `cloudplus-prod-data`. The dev one is destroyed because the resource name is parameterised — typical IaC behaviour.

---

## Step 7 — Cleanup

```bash
terraform destroy -auto-approve
docker rm -f localstack
```

---

## Test it

Run these checks to prove the lab worked before you move on:

```bash
cd /tmp/tf && terraform state list
terraform state show aws_s3_bucket.data | head -10
terraform plan -detailed-exitcode
docker exec localstack awslocal s3 ls
git -C /tmp/tf log --oneline
```

**Expected:** Run this before Step 7. `terraform state list` prints `aws_s3_bucket.data`; `state show` reports the bucket name (`cloudplus-prod-data` after Step 6); `terraform plan` reports **No changes. Your infrastructure matches the configuration** (exit code 0) once applied, or lists the drifted tag if you run it before re-applying; `awslocal s3 ls` shows the bucket really exists in LocalStack; and `git log` shows the `initial infra` commit.

---

## What you learned
- IaC: declarative config → cloud resources.
- Plan before apply.
- Drift detection catches manual changes.
- State must be stored safely.

## Free tools used
- Terraform — https://developer.hashicorp.com/terraform
- LocalStack Community — https://www.localstack.cloud
- OpenTofu (Terraform fork) — https://opentofu.org

---

## Files in this lab

| File | Purpose |
|------|---------|
| [`main.tf`](main.tf) | Step 2 Terraform config — LocalStack AWS provider plus the parameterised S3 bucket. |
