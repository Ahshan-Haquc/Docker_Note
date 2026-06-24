# Docker Image Docker Hub এ Push করার সম্পূর্ণ গাইড

## ভূমিকা

যখন আমরা Local Machine এ Docker Image তৈরি করি, তখন সেটি শুধুমাত্র আমাদের কম্পিউটারেই থাকে। কিন্তু যদি আমরা সেই Image অন্য কোনো Developer, VPS অথবা Production Server এ ব্যবহার করতে চাই, তাহলে Image টি Docker Hub এ Upload (Push) করতে হয়।

Docker Hub হচ্ছে Docker Image সংরক্ষণ করার একটি Cloud Registry।

---

## Step 1: Docker Hub Account তৈরি করা

প্রথমে Docker Hub এ একটি Account তৈরি করতে হবে।

এরপর Docker Desktop এ Login করতে হবে অথবা Terminal থেকে Login করতে পারো।

```bash
docker login
```

Command চালানোর পর:

```text
Username: তোমার Docker Hub Username

Password: তোমার Docker Hub Password
```

যদি Login সফল হয়:

```text
Login Succeeded
```

---

## Step 2: Local Images দেখা

প্রথমে তোমার Local Machine এ কোন কোন Image আছে তা দেখো।

```bash
docker images
```

উদাহরণ:

```text
REPOSITORY       TAG       IMAGE ID

my-app           latest    abc123efg456
```

এখানে,

* Repository Name = my-app
* Tag = latest

---

## Step 3: Docker Hub Username অনুযায়ী Image Tag করা

Docker Hub এ Push করার আগে Image এর নাম Docker Hub Username সহ Tag করতে হবে।

Syntax:

```bash
docker tag local-image-name dockerhub-username/repository-name:tag
```

উদাহরণ:

```bash
docker tag my-app rashed/my-app:latest
```

এখন আবার:

```bash
docker images
```

Output:

```text
REPOSITORY

my-app

rashed/my-app
```

দুটো Image একই Image কে নির্দেশ করে, শুধু নাম আলাদা।

---

## Step 4: Docker Hub এ Image Push করা

এখন Image Push করো:

```bash
docker push rashed/my-app:latest
```

Docker Image এর Layers গুলো Upload হতে শুরু করবে।

Output:

```text
Pushing

Layer already exists

latest: digest: sha256:xxxxx

status: pushed
```

এর মানে Image সফলভাবে Docker Hub এ Upload হয়েছে।

---

## Step 5: Docker Hub এ Image Verify করা

Docker Hub Website এ Login করো।

Repositories Section এ গেলে দেখতে পাবে:

```text
rashed/my-app
```

এখন পৃথিবীর যেকোনো জায়গা থেকে এই Image Pull করা যাবে।

---

## অন্য Computer বা VPS এ Image Pull করা

Syntax:

```bash
docker pull username/repository:tag
```

উদাহরণ:

```bash
docker pull rashed/my-app:latest
```

তারপর Container Run করো:

```bash
docker run -d -p 5000:5000 rashed/my-app:latest
```

---

## MERN Stack Developer এর বাস্তব Workflow

ধরো তোমার একটি Express Backend Project আছে।

### Step 1

Docker Image Build:

```bash
docker build -t backend .
```

---

### Step 2

Image Run:

```bash
docker run -d -p 5000:5000 backend
```

---

### Step 3

Docker Hub এর জন্য Tag:

```bash
docker tag backend rashed/backend:latest
```

---

### Step 4

Docker Hub এ Push:

```bash
docker push rashed/backend:latest
```

---

### Step 5

VPS এ Pull:

```bash
docker pull rashed/backend:latest
```

---

### Step 6

VPS এ Run:

```bash
docker run -d -p 5000:5000 rashed/backend:latest
```

---

## সংক্ষেপে পুরো Flow

```text
Local Project

↓

docker build

↓

Local Image

↓

docker tag

↓

docker login

↓

docker push

↓

Docker Hub

↓

docker pull

↓

VPS

↓

docker run

↓

Application Live
```

---

## সবচেয়ে গুরুত্বপূর্ণ Commands

Login:

```bash
docker login
```

Images দেখা:

```bash
docker images
```

Image Tag:

```bash
docker tag my-app username/my-app:latest
```

Push:

```bash
docker push username/my-app:latest
```

Pull:

```bash
docker pull username/my-app:latest
```

Run:

```bash
docker run -d -p 5000:5000 username/my-app:latest
```

---

## Beginner দের সাধারণ ভুল

1. docker login না করে push করা।

```text
denied: requested access to the resource is denied
```

2. Username ভুল লেখা।

```text
docker push my-app
```

এটা ভুল।

সঠিক:

```text
docker push username/my-app:latest
```

3. Tag না দিয়ে Push করা।

প্রথমে:

```bash
docker tag local-image username/image:latest
```

তারপর Push করতে হবে।

---

## Final Goal

একজন MERN Stack Developer হিসেবে তোমার Production Workflow সাধারণত এরকম হবে:

```text
React/Node Project

↓

Dockerfile

↓

docker build

↓

Docker Image

↓

Docker Hub

↓

Ubuntu VPS

↓

docker pull

↓

docker run

↓

Nginx Reverse Proxy

↓

Production Deployment
```

এই Workflow টি আয়ত্ত করতে পারলে তুমি Local Machine থেকে Production VPS পর্যন্ত সম্পূর্ণ Docker Deployment নিজে করতে পারবে।
