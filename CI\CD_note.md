# CI/CD with GitHub Actions — VPS-এ Auto Deploy (বাংলা)

## Tutorial link
https://www.youtube.com/watch?v=LQEYDJSyW8M&list=LL&index=1&t=588s

## ভূমিকা: CI/CD মানে কী?

আগের গাইডে আমরা manually deploy করেছিলাম:
```bash
ssh ahsan@41.112.107.1
cd /home/ahsan/apps/your-project
git pull origin main
docker compose -f docker-compose.prod.yml up -d --build
```

প্রতিবার code বদলালে এই ৪টা command নিজে হাতে চালাতে হচ্ছে। **CI/CD** এই পুরো কাজটা automate করে দেয়:

```
আপনি শুধু এটা করবেন:
  git push origin main

এরপর automatically ঘটবে:
  ┌─────────────┐    ┌──────────────┐    ┌─────────────────┐
  │ GitHub       │───▶│ GitHub       │───▶│ VPS-এ SSH করে   │
  │ push পায়    │    │ Actions চালু │    │ pull + rebuild  │
  └─────────────┘    └──────────────┘    └─────────────────┘
                                                  │
                                                  ▼
                                          Website আপডেট হয়ে গেছে!
```

**CI/CD-এর পূর্ণরূপ:**
- **CI (Continuous Integration)** — code push হলে automatically test/build চালানো
- **CD (Continuous Deployment)** — সেই build automatically server-এ deploy করা

**GitHub Actions** হলো GitHub-এর নিজস্ব CI/CD টুল — কোনো বাইরের সার্ভিস লাগে না, GitHub-এর মধ্যেই সব হয়।

---

## কীভাবে কাজ করে — পুরো চিত্র

```
┌──────────────────────────────────────────────────────────────────────────┐
│ আপনার Laptop                                                            │
│                                                                          │
│   code লিখলেন → git push origin main                                   │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ GitHub Repository                                                         │
│                                                                          │
│   main branch-এ নতুন commit এলো                                        │
│   → .github/workflows/deploy.yml ফাইল trigger হলো                      │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ GitHub Actions Runner (GitHub-এর নিজস্ব cloud মেশিন, বিনামূল্যে)      │
│                                                                          │
│   ১. Repository checkout করে                                           │
│   ২. SSH key দিয়ে VPS-এ connect করে                                   │
│   ৩. VPS-কে command পাঠায়: "git pull করো, rebuild করো"               │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │ SSH (port 22)
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ Hostinger VPS (41.112.107.1)                                            │
│                                                                          │
│   git pull origin main                                                  │
│   docker compose -f docker-compose.prod.yml up -d --build              │
│   → নতুন version live!                                                  │
└──────────────────────────────────────────────────────────────────────────┘
```

মূল কথা: **GitHub Actions নিজে কিছু build করে না, বরং VPS-কে নির্দেশ পাঠায় — "তুমি pull করে rebuild করো"।**

---

## ধাপ ১: SSH Key তৈরি করা (GitHub → VPS Passwordless Access)

GitHub Actions-কে VPS-এ login করতে হলে password ছাড়া, **SSH key** দিয়ে authenticate করতে হবে — কারণ automated process-এ কেউ বসে password টাইপ করবে না।

### কেন আলাদা একটা SSH Key দরকার?

আপনার নিজের কম্পিউটারের SSH key VPS-এ login করার জন্য ব্যবহার হয়। কিন্তু GitHub Actions-এর জন্য **আলাদা একটা dedicated key** তৈরি করা best practice — এতে যদি কোনোভাবে এই key leak হয়, শুধু deployment access-ই risk-এ পড়বে, আপনার personal access না।

### আপনার নিজের Laptop-এ (VPS-এ না!) এই কাজ করুন

```bash
# একটা নতুন SSH key জোড়া তৈরি করুন, শুধু deployment-এর জন্য
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github-actions-deploy

# কোনো passphrase দেবেন না — শুধু Enter চাপুন (automated process-এ passphrase দেওয়া যায় না)
```

এতে দুটো ফাইল তৈরি হবে:
```
~/.ssh/github-actions-deploy        ← Private key (গোপন, কাউকে দেবেন না)
~/.ssh/github-actions-deploy.pub    ← Public key (VPS-এ বসাতে হবে)
```

### Public Key VPS-এ যোগ করুন

```bash
# Public key-এর content দেখুন
cat ~/.ssh/github-actions-deploy.pub
```

Output এরকম দেখাবে:
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIxxxxxxxxxxxxxxxxxxxx github-actions-deploy
```

এই পুরো লাইনটা copy করুন। এখন VPS-এ connect করুন:

```bash
ssh ahsan@41.112.107.1
```

VPS-এর ভিতরে:
```bash
# authorized_keys ফাইলে public key যোগ করুন
nano ~/.ssh/authorized_keys
```

ফাইলের **শেষে নতুন লাইনে** copy করা public key paste করুন (আগের কোনো key থাকলে মুছবেন না)। Save করুন: `Ctrl+X` → `Y` → `Enter`

```bash
# Permission ঠিক আছে কিনা নিশ্চিত করুন
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

### Private Key Test করুন

আপনার laptop-এ ফিরে এসে test করুন এই নতুন key দিয়ে login হয় কিনা:

```bash
ssh -i ~/.ssh/github-actions-deploy ahsan@41.112.107.1
```

Password ছাড়াই login হয়ে গেলে — সফল! এখন `exit` লিখে বের হয়ে আসুন।

---

## ধাপ ২: GitHub Secrets সেট করা

Private key, VPS IP, ইত্যাদি sensitive তথ্য কখনো সরাসরি code-এ লেখা যাবে না। GitHub-এর **Secrets** ফিচার ব্যবহার করে এগুলো নিরাপদে রাখা হয়।

### GitHub Repository-তে Secrets যোগ করুন

```
GitHub repository খুলুন
→ Settings (repo-এর, account-এর না)
→ Secrets and variables → Actions
→ "New repository secret" বাটনে ক্লিক করুন
```

নিচের secrets গুলো একে একে যোগ করুন:

| Secret Name | Value | কোথা থেকে পাবেন |
|---|---|---|
| `VPS_HOST` | `41.112.107.1` | আপনার VPS IP |
| `VPS_USERNAME` | `ahsan` | VPS-এর user নাম |
| `VPS_SSH_KEY` | Private key-এর পুরো content | `cat ~/.ssh/github-actions-deploy` |
| `VPS_PORT` | `22` | SSH default port |

### `VPS_SSH_KEY` কীভাবে copy করবেন

```bash
# Laptop-এ এই command চালান
cat ~/.ssh/github-actions-deploy
```

Output (পুরোটা, `-----BEGIN` থেকে `-----END` পর্যন্ত সব) copy করুন:
```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZWQy
NTUxOQAAACBxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
...
-----END OPENSSH PRIVATE KEY-----
```

এই পুরোটা `VPS_SSH_KEY` secret-এর value হিসেবে paste করুন।

> ⚠️ **গুরুত্বপূর্ণ:** Private key (যেটা `.pub` দিয়ে শেষ হয় না) কখনো কারো সাথে শেয়ার করবেন না, কখনো git-এ commit করবেন না। শুধু GitHub Secrets-এ paste করুন।

---

## ধাপ ৩: GitHub Actions Workflow ফাইল তৈরি করা

এখন আপনার project-এ (laptop-এ, VS Code দিয়ে) একটা workflow ফাইল তৈরি করবেন।

### Folder Structure

```
your-project/
├── .github/
│   └── workflows/
│       └── deploy.yml      ← এই ফাইলটা তৈরি করতে হবে
├── docker-compose.prod.yml
├── Dockerfile
└── ... (বাকি project files)
```

### `.github/workflows/deploy.yml`

```yaml
name: Deploy to VPS

# ── কখন এই workflow চলবে ──────────────────────────────────────────────────
on:
  push:
    branches:
      - main          # শুধু main branch-এ push হলেই চলবে

# ── কী কাজ করবে ───────────────────────────────────────────────────────────
jobs:
  deploy:
    runs-on: ubuntu-latest    # GitHub-এর cloud মেশিনে চলবে (Ubuntu)

    steps:
      # ধাপ ১: Repository checkout করা (যদিও আমরা VPS-এ direct SSH করছি,
      # ভবিষ্যতে build/test step যোগ করলে এটা দরকার হবে)
      - name: Checkout repository
        uses: actions/checkout@v4

      # ধাপ ২: SSH দিয়ে VPS-এ connect করে deploy command চালানো
      - name: Deploy to VPS via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USERNAME }}
          key: ${{ secrets.VPS_SSH_KEY }}
          port: ${{ secrets.VPS_PORT }}
          script: |
            cd /home/ahsan/apps/your-project  #my project patch , pwd
            git pull origin main
            docker compose -f docker-compose.prod.yml up -d --build
            docker image prune -f
```

### প্রতিটা অংশের ব্যাখ্যা

**`on: push: branches: - main`**
```yaml
on:
  push:
    branches:
      - main
```
এটাই trigger condition। শুধু `main` branch-এ push হলে workflow চলবে। অন্য branch (`dev`, `feature/x`) এ push করলে চলবে না।

**`runs-on: ubuntu-latest`**
GitHub Actions আপনার জন্য একটা সাময়িক (temporary) Ubuntu মেশিন তৈরি করে, যেখানে এই steps চলে। কাজ শেষে এই মেশিন মুছে যায়। এটা সম্পূর্ণ বিনামূল্যে (public repo-এ unlimited, private repo-এ মাসে 2000 মিনিট free)।

**`actions/checkout@v4`**
এটা GitHub-এর official action — repository-র code এই temporary মেশিনে নিয়ে আসে। আমরা সরাসরি build করছি না, কিন্তু এটা রাখা ভালো practice (পরে দরকার হতে পারে)।

**`appleboy/ssh-action@v1.0.3`**
এটা একটা popular third-party action যা SSH দিয়ে কোনো remote server-এ command চালাতে দেয়। GitHub Marketplace-এ হাজার হাজার এমন ready-made action আছে।

**`${{ secrets.VPS_HOST }}`**
এভাবে আগে সেট করা GitHub Secret গুলো ব্যবহার করা হয়। `secrets.VPS_HOST` মানে Secrets-এ সেট করা `VPS_HOST` এর value।

**`script: |`**
এই `|` symbol মানে multi-line string। এর নিচে যা লেখা, তা ঠিক সেভাবেই VPS-এর terminal-এ চলবে — যেন আপনি নিজে SSH করে এই command গুলো টাইপ করছেন।

---

## ধাপ ৪: Workflow Push করা ও Test করা

```bash
# Laptop-এ, project folder-এ
git add .github/workflows/deploy.yml
git commit -m "Add CI/CD pipeline for auto deploy"
git push origin main
```

**এই push-টাই প্রথম deployment trigger করবে!**

### Workflow চলছে কিনা দেখুন

```
GitHub repository → "Actions" ট্যাব
→ "Deploy to VPS" workflow দেখবেন
→ ক্লিক করে live log দেখুন
```

প্রতিটা step-এর পাশে ✅ (সফল) বা ❌ (ব্যর্থ) দেখাবে। কোনো ধাপে সমস্যা হলে সেই step-এ ক্লিক করে error message দেখা যায়।

সফল হলে দেখাবে:
```
✅ Checkout repository
✅ Deploy to VPS via SSH
   Run appleboy/ssh-action@v1.0.3
   ======CMD======
   cd /home/ahsan/apps/your-project
   git pull origin main
   docker compose -f docker-compose.prod.yml up -d --build
   ======END======
   ==============================================
   ✅ Successfully executed commands to all hosts.
```

---

## একটু Advanced: Docker Hub দিয়ে Build (পদ্ধতি ২)

উপরের পদ্ধতিতে VPS নিজেই `docker compose build` করে — অর্থাৎ VPS-এর CPU/RAM ব্যবহার হয় build করতে। ছোট VPS (KVM-1, 4GB RAM)-এ বড় project build করতে গেলে কখনো কখনো মেমোরি কম পড়তে পারে।

**বিকল্প পদ্ধতি:** GitHub Actions-এর শক্তিশালী cloud মেশিনে build করে, তৈরি image **Docker Hub**-এ push করে, VPS শুধু সেই ready image **pull** করে।

```
GitHub Actions Runner          Docker Hub              VPS
──────────────────────         ───────────              ───
docker build                    
docker push        ────────▶   Image জমা থাকে  ────▶  docker pull
                                                        docker compose up -d
                                (VPS-এ build করতে হয় না, শুধু pull করে)
```

### `.github/workflows/deploy.yml` (Docker Hub সহ)

```yaml
name: Build and Deploy to VPS

on:
  push:
    branches:
      - main

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/my-node-app:latest

  deploy:
    needs: build-and-push    # আগের job সফল হলেই এটা চলবে
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to VPS via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USERNAME }}
          key: ${{ secrets.VPS_SSH_KEY }}
          port: ${{ secrets.VPS_PORT }}
          script: |
            cd /home/ahsan/apps/your-project
            docker pull ${{ secrets.DOCKERHUB_USERNAME }}/my-node-app:latest
            docker compose -f docker-compose.prod.yml up -d
            docker image prune -f
```

**নতুন Secrets দরকার হবে:**
| Secret Name | Value |
|---|---|
| `DOCKERHUB_USERNAME` | আপনার Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub-এ Account Settings → Security → New Access Token |

**`needs: build-and-push`** — এটা বলে দেয় `deploy` job তখনই চলবে যখন `build-and-push` job সম্পূর্ণ সফল হবে। এটা একটা **pipeline** তৈরি করে — ধাপে ধাপে কাজ।

**কখন কোন পদ্ধতি বেছে নেবেন?**

| পদ্ধতি | কখন ব্যবহার করবেন |
|---|---|
| পদ্ধতি ১ (VPS-এ build) | ছোট-মাঝারি project, simple setup চাইলে |
| পদ্ধতি ২ (Docker Hub) | VPS-এর RAM কম, build দ্রুত করতে চাইলে, বড় team |

আপনার KVM-1 (4GB RAM)-এর জন্য সাধারণত **পদ্ধতি ১** যথেষ্ট, যদি না build প্রক্রিয়া খুব heavy হয়।

---

## একটা গুরুত্বপূর্ণ বিষয়: `.env` File CI/CD-তে কীভাবে Handle করবেন

`.env` ফাইল git-এ থাকে না (এটা `.gitignore`-এ), তাই `git pull` করলে এটা VPS-এ আসবে না। এটা ঠিক আছে — `.env` ফাইল **VPS-এই একবার manually তৈরি করে রেখে দিতে হবে** (আগের গাইডে যেভাবে করেছিলাম)।

```bash
# এটা VPS-এ একবারই করতে হবে, deploy-এর সময় বারবার না
ssh ahsan@41.112.107.1
cd /home/ahsan/apps/your-project
nano .env
# মান গুলো বসান, save করুন
```

CI/CD pipeline প্রতিবার শুধু `git pull` ও `docker compose up --build` করবে — `.env` ফাইল VPS-এই থেকে যাবে, প্রতিবার নতুন করে লাগবে না।

**যদি `.env` value বদলাতে হয়** (নতুন secret যোগ, password বদল), তখন VPS-এ গিয়ে manually `nano .env` দিয়ে edit করতে হবে — এটা automate করা risky (Secrets file এমনিতেই sensitive)।

---

## সাধারণ ভুলগুলো

### ভুল ১: Private Key-এর বদলে Public Key Secrets-এ দেওয়া
```
❌ VPS_SSH_KEY = ssh-ed25519 AAAA... (এটা .pub ফাইলের content)
✅ VPS_SSH_KEY = -----BEGIN OPENSSH PRIVATE KEY----- ... (private key, .pub ছাড়া ফাইল)
```
Public key VPS-এ `authorized_keys`-এ থাকে, Private key GitHub Secrets-এ থাকে — উল্টে ফেললে SSH authentication fail করবে।

### ভুল ২: `main` Branch-এর নাম ভুল ধরে নেওয়া
```yaml
# আপনার repo-র default branch "master" হলে এটা কখনো trigger হবে না
on:
  push:
    branches:
      - main    # ❌ যদি আপনার branch "master" হয়

# GitHub repo-তে branch নাম চেক করুন, তারপর ঠিক করুন
on:
  push:
    branches:
      - master  # ✅
```

### ভুল ৩: VPS-এ Project Path ভুল দেওয়া
```yaml
script: |
  cd /home/ahsan/apps/your-project   # এই path টা ঠিক আছে কিনা VPS-এ গিয়ে নিশ্চিত করুন
```
`pwd` এবং `ls` দিয়ে VPS-এ সঠিক path verify করে নিন।

### ভুল ৪: SSH Key-তে Passphrase দেওয়া
```bash
# ❌ Passphrase দিলে automated SSH login কাজ করবে না
ssh-keygen -t ed25519 -f ~/.ssh/github-actions-deploy
# Enter passphrase: ****  ← এটা ফাঁকা রাখুন!

# ✅ Passphrase ছাড়া (শুধু Enter চাপুন)
```

### ভুল ৫: Workflow ফাইলের YAML Indentation ভুল করা
```yaml
# ❌ Indentation ভুল — YAML strict spacing মানে
jobs:
deploy:           # ভুল indentation
runs-on: ubuntu-latest

# ✅ সঠিক indentation (২ space consistently)
jobs:
  deploy:
    runs-on: ubuntu-latest
```
YAML-এ Tab না ব্যবহার করে সবসময় Space ব্যবহার করুন।

### ভুল ৬: GitHub Actions Secrets না দিয়ে সরাসরি IP/Password লেখা
```yaml
# ❌ ভয়ঙ্কর — IP/key public repo-তে দেখা যাবে
host: 41.112.107.1
key: -----BEGIN OPENSSH PRIVATE KEY-----...

# ✅ সবসময় secrets ব্যবহার করুন
host: ${{ secrets.VPS_HOST }}
key: ${{ secrets.VPS_SSH_KEY }}
```

---

## Workflow Test ও Debug করা

### Local-এ Test করা সম্ভব না — তাই সতর্কতার সাথে এগোন

```bash
# ছোট পরিবর্তন push করে দেখুন pipeline ঠিকঠাক চলে কিনা
echo "// test comment" >> src/server.js
git add .
git commit -m "test: trigger CI/CD pipeline"
git push origin main
```

GitHub Actions ট্যাবে গিয়ে দেখুন:
```
GitHub repo → Actions → সাম্প্রতিক workflow run → ক্লিক
→ "deploy" job → প্রতিটা step expand করে log দেখুন
```

### সাধারণ Error গুলো

| Error Message | কারণ | সমাধান |
|---|---|---|
| `Permission denied (publickey)` | Public key VPS-এ সঠিকভাবে যোগ হয়নি | `authorized_keys` আবার চেক করুন |
| `Host key verification failed` | প্রথমবার SSH-এ known_hosts সমস্যা | Action-এ `fingerprint` বা `--StrictHostKeyChecking=no` যোগ করুন |
| `No such file or directory` | VPS-এ path ভুল | `cd` command-এর path VPS-এ গিয়ে নিশ্চিত করুন |
| `docker: command not found` | SSH non-interactive shell-এ PATH সমস্যা | Script-এ `export PATH=$PATH:/usr/bin` যোগ করুন বা full path (`/usr/bin/docker`) ব্যবহার করুন |

---

## সম্পূর্ণ Flow — একনজরে

```
┌──────────────────────────────────────────────────────────────────────────┐
│ একবারের Setup                                                           │
│                                                                          │
│ 1. Laptop-এ নতুন SSH key জোড়া তৈরি (ssh-keygen)                      │
│ 2. Public key VPS-এর authorized_keys-এ যোগ                            │
│ 3. GitHub repo → Settings → Secrets-এ ৪টা secret যোগ                  │
│    (VPS_HOST, VPS_USERNAME, VPS_SSH_KEY, VPS_PORT)                     │
│ 4. .github/workflows/deploy.yml ফাইল তৈরি                              │
│ 5. VPS-এ .env ফাইল manually তৈরি (একবার)                              │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│ প্রতিদিনের কাজ (এখন থেকে এতটুকুই!)                                    │
│                                                                          │
│  git add .                                                               │
│  git commit -m "feature: added new endpoint"                           │
│  git push origin main                                                   │
│                                                                          │
│  ↓ স্বয়ংক্রিয়ভাবে ↓                                                  │
│                                                                          │
│  GitHub Actions → SSH → VPS → git pull → docker compose up --build     │
│  → ✅ Website আপডেট হয়ে গেছে                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```bash
# ── SSH Key তৈরি (Laptop-এ) ──────────────────────────────────────────────
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github-actions-deploy
cat ~/.ssh/github-actions-deploy.pub     # VPS-এ যোগ করতে
cat ~/.ssh/github-actions-deploy         # GitHub Secrets-এ যোগ করতে

# ── VPS-এ Public Key যোগ ─────────────────────────────────────────────────
nano ~/.ssh/authorized_keys              # নতুন লাইনে paste

# ── GitHub Secrets (Settings → Secrets and variables → Actions) ─────────
VPS_HOST       = 41.112.107.1
VPS_USERNAME   = ahsan
VPS_SSH_KEY    = (private key পুরোটা)
VPS_PORT       = 22

# ── Workflow ফাইল Location ───────────────────────────────────────────────
.github/workflows/deploy.yml

# ── Deploy Trigger করা ────────────────────────────────────────────────────
git push origin main          # ব্যস, এতটুকুই!
```

---

## মনে রাখার সহজ নিয়ম

```
CI/CD = Automated "ssh + git pull + docker compose up"

GitHub Actions = GitHub-এর built-in robot,
                 যে আপনার হয়ে SSH করে VPS-এ command চালায়

SSH Key জোড়া:
  Private key (VPS_SSH_KEY)  → GitHub Secrets-এ থাকে (গোপন)
  Public key                  → VPS-এর authorized_keys-এ থাকে

Trigger:
  git push origin main  →  workflow চলে  →  VPS আপডেট হয়
```

এই pipeline একবার ঠিকভাবে সেট করলে, এরপর থেকে শুধু `git push` করলেই আপনার Hostinger VPS-এ website automatically আপডেট হয়ে যাবে — manual SSH করার আর দরকার নেই।
