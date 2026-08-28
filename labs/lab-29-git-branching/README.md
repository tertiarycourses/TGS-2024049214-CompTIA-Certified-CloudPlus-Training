# Lab 29 — Git Source Control & Branching

In this lab you will exercise every CV0-004 source-control verb: **commit, push, branch, merge, pull request review** — using a local Gitea server as your "GitHub clone".

## Lab platform

Run all commands on the **Killercoda Ubuntu Playground**:

https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — Install git and run Gitea (self-hosted)

```bash
apt update && apt install -y git docker.io curl jq
systemctl start docker

docker exec -u git gitea gitea migrate
```
```
docker exec gitea mkdir -p /data/gitea/conf
```
```
docker exec gitea sh -c 'cat > /data/gitea/conf/app.ini <<EOF
[database]
DB_TYPE = sqlite3
PATH = /data/gitea/gitea.db

[server]
DOMAIN = 172.30.1.2
HTTP_PORT = 3000
ROOT_URL = http://172.30.1.2:3000/
SSH_PORT = 2222

[security]
INSTALL_LOCK = true

[service]
DISABLE_REGISTRATION = false
REQUIRE_SIGNIN_VIEW = false

[log]
MODE = console
EOF'
```
```
docker restart gitea
sleep 10
```
#Create admin repo
```
docker exec -u git gitea \
  gitea admin user create \
  --username admin \
  --password cloudplus \
  --email admin@example.com \
  --admin
```
#Verify
```
curl -s -u admin:cloudplus \
  http://172.30.1.2:3000/api/v1/user | jq '.login'
```
#create infra
```
curl -s -u admin:cloudplus \
  -X POST http://172.30.1.2:3000/api/v1/user/repos \
  -H 'Content-Type: application/json' \
  -d '{"name":"infra","auto_init":true,"default_branch":"main"}' | jq '.full_name'
```



---

## Step 5 — Open a Pull Request

```bash
cd /root
git clone http://172.30.1.2:3000/admin/infra.git
cd infra

git config user.name "admin"
git config user.email "admin@example.com"

cat > main.tf <<'EOF'
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
    }
  }
}

provider "local" {}

resource "local_file" "demo" {
  filename = "${path.module}/hello.txt"
  content  = "Hello from Terraform + Gitea"
}
EOF

git add .
git commit -m "Add initial Terraform configuration"
git branch -M main
git push origin main
```

Open the PR in the UI and review the diff — that is the **code review** sub-objective.

---

## Step 6 — Test GitOps workflow

```bash
echo "GitOps change" >> hello.txt

git add hello.txt
git commit -m "Update infrastructure configuration"
git push origin main
```
#verify Gitea's API:

```
curl -s -u admin:cloudplus \
  http://172.30.1.2:3000/api/v1/repos/admin/infra/commits | jq '.[0].commit.message'

```

## Step 9 — Cleanup

```bash
docker rm -f gitea
rm -rf /tmp/work
```

---

## Test it

Run these checks to prove the lab worked before you move on:

```bash
curl -sI http://localhost:3000 | head -1
curl -s -u admin:cloudplus http://localhost:3000/api/v1/repos/admin/infra/branches | jq '.[].name'
curl -s -u admin:cloudplus http://localhost:3000/api/v1/repos/admin/infra/pulls?state=all | jq '.[] | {title, state}'
cd /tmp/work/infra && git log --oneline --graph --all | head -15
grep -c '<<<<<<<' main.tf
```

**Expected:** Run this before Step 9. Gitea returns `HTTP/1.1 200 OK`; the branch list contains `main`, `feature/encryption` and `feature/region`; the PR query shows `{"title":"Add SSE","state":"closed"}` — closed because it was merged; the commit graph shows the feature branches joining `main` at a merge commit; and `grep -c` returns `0`, proving the conflict markers were resolved before the final commit.

---

## What you learned
- Commit → push → branch → PR → merge.
- Conflict resolution is a normal Git workflow.
- Branch protection enforces the review policy.

## Free tools used
- git (built-in) — https://git-scm.com
- Gitea — https://about.gitea.com
- GitHub (free public repos) — https://github.com
- GitLab Community — https://about.gitlab.com
- Pro Git book (free) — https://git-scm.com/book

---

## Files in this lab

| File | Purpose |
|------|---------|
| [`main.tf`](main.tf) | Step 3 initial Terraform file committed to the Gitea repo. |
| [`encryption.tf.snippet`](encryption.tf.snippet) | Step 4 block appended to `main.tf` on the `feature/encryption` branch. |
