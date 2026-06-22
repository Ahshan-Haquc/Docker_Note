# 🐳 Chapter 3: Docker Basics Commands

> **ভাষা:** বাংলা | **বিষয়:** Docker-এর সবচেয়ে গুরুত্বপূর্ণ কমান্ডগুলোর বিস্তারিত আলোচনা

---

## 📋 সূচিপত্র

| # | কমান্ড | কাজ |
|---|--------|-----|
| 1 | [`docker pull`](#১-docker-pull--image-ডাউনলোড-করা) | Registry থেকে Image ডাউনলোড |
| 2 | [`docker images`](#২-docker-images--ডাউনলোড-হওয়া-images-দেখা) | লোকাল Images-এর তালিকা |
| 3 | [`docker run`](#৩-docker-run--container-তৈরি-করে-চালানো) | Container তৈরি ও চালু |
| 4 | [`docker run -it`](#৪-docker-run--it--interactive-terminal-এ-container-চালানো) | Interactive Shell-এ Container |
| 5 | [`docker ps`](#৫-docker-ps--চলমান-container-দেখা) | চলমান Container দেখা |
| 6 | [`docker ps -a`](#৬-docker-ps--a--সব-container-দেখা) | সব Container দেখা |
| 7 | [`docker start`](#৭-docker-start--বন্ধ-container-আবার-চালু-করা) | বন্ধ Container চালু |
| 8 | [`docker stop`](#৮-docker-stop--চলমান-container-বন্ধ-করা) | চলমান Container বন্ধ |
| 9 | [`docker rm`](#৯-docker-rm--container-মুছে-ফেলা) | Container মুছে ফেলা |
| 10 | [`docker rmi`](#১০-docker-rmi--image-মুছে-ফেলা) | Image মুছে ফেলা |
| 11 | [`docker logs`](#১১-docker-logs--container-এর-outputlog-দেখা) | Container-এর Log দেখা |
| 12 | [`docker exec`](#১২-docker-exec--চলমান-container-এর-ভিতরে-কমান্ড-চালানো) | Container-এ কমান্ড চালানো |

---

## ১. `docker pull` — Image ডাউনলোড করা

### 🔍 কী করে?
`docker pull` কমান্ডটি Docker Hub (অথবা অন্য কোনো private/public registry) থেকে একটি image আপনার লোকাল মেশিনে ডাউনলোড করে আনে। এটা ঠিক যেভাবে আপনি একটা `.zip` ফাইল ইন্টারনেট থেকে ডাউনলোড করেন, ঠিক সেভাবেই।

### ❓ কেন দরকার?
Container চালাতে হলে আগে একটা image দরকার। সেই image হয়তো আপনার মেশিনে নেই — তাই সেটাকে registry থেকে আনতে হয়। অনেক সময় আমরা `docker run` করলেই image auto-pull হয়ে যায়, কিন্তু production বা CI/CD pipeline-এ আমরা চাই আগে থেকেই pull করে রাখতে, যাতে deployment-এর সময় network delay না হয়।

### 📅 কখন ব্যবহার করব?
- নতুন কোনো software/database/runtime ব্যবহার করার আগে (যেমন nginx, mysql, node)
- কোনো image-এর নির্দিষ্ট version বা latest update আনতে চাইলে
- Server pre-warm করতে চাইলে (deploy করার আগেই image রেডি রাখা)

### 📝 Syntax

```bash
docker pull [OPTIONS] NAME[:TAG]
```

**প্রতিটি অংশের ব্যাখ্যা:**

| অংশ | ব্যাখ্যা |
|-----|----------|
| `docker` | Docker CLI tool কে call করা হচ্ছে |
| `pull` | সাব-কমান্ড, registry থেকে download করার নির্দেশ |
| `NAME` | image-এর নাম, যেমন nginx, ubuntu, node, mysql |
| `:TAG` | ঐচ্ছিক, image-এর version বোঝায় (যেমন `:18`, `:1.25`, `:alpine`)। না দিলে Docker ধরে নেয় `:latest` |
| `OPTIONS` | যেমন `-a` (সব tag pull করা), `--platform` (নির্দিষ্ট architecture-এর জন্য) |

### 💡 উদাহরণ

```bash
# সবচেয়ে সাধারণ - latest version pull করা
docker pull nginx

# নির্দিষ্ট version pull করা (production-এ সবসময় এটা করা উচিত)
docker pull nginx:1.25

# lightweight version (alpine based, ছোট সাইজ)
docker pull node:18-alpine

# নির্দিষ্ট OS version
docker pull ubuntu:22.04
```

### 🖥️ Output ও তার ব্যাখ্যা

```
$ docker pull nginx:1.25
1.25: Pulling from library/nginx
a378f10b3218: Pull complete
1bbf4c87c750: Pull complete
8c7e6e4adf57: Pull complete
Digest: sha256:1b87c...
Status: Downloaded newer image for nginx:1.25
docker.io/library/nginx:1.25
```

- প্রতিটি লাইন (`a378f10b3218: Pull complete`) একটা image **layer** — Docker image-কে কয়েকটা layer-এ ভাগ করে রাখে, যাতে একই layer বারবার download করতে না হয়।
- `Status: Downloaded newer image` মানে নতুন করে ডাউনলোড হয়েছে। যদি আগে থেকেই থাকে, লেখা আসবে `Image is up to date`।
- শেষ লাইনে পূর্ণ image নাম (`docker.io/library/nginx:1.25`) দেখায়, registry সহ।

### ⚠️ সাধারণ ভুল

> **Tag না দেওয়া** — শুধু `docker pull nginx` লিখলে `:latest` আসবে, যা production-এ অনিশ্চিত (আজ এক version, কাল আরেক version হতে পারে)। সবসময় নির্দিষ্ট version pin করুন।

> **Image নাম বানান ভুল করা** — `nginex` লিখলে "not found" error আসবে।

> **Pull না করেই run করা যায়, তাই pull-কে অপ্রয়োজনীয় মনে করা** — আসলে run নিজে থেকেও pull করে, কিন্তু explicit pull production deployment-এ predictable behavior দেয়।

> **Disk space খেয়াল না রাখা** — বড় বড় image (যেমন full Ubuntu বা full Node) বারবার pull করলে অনেক জায়গা নষ্ট হয়।

### 🔄 Workflow-এ অবস্থান
এটাই প্রথম ধাপ। Image ছাড়া container বানানো সম্ভব না। এরপরের কাজ হলো — সেই image আমার মেশিনে আদৌ এসেছে কিনা, তা `docker images` দিয়ে যাচাই করা।

---

## ২. `docker images` — ডাউনলোড হওয়া Images দেখা

### 🔍 কী করে?
লোকাল মেশিনে যত image ডাউনলোড করা আছে, তার একটা তালিকা দেখায়।

### ❓ কেন দরকার?
- কোন কোন image আপনার মেশিনে আছে তা জানতে
- প্রতিটা image কত জায়গা নিচ্ছে তা দেখতে
- ভুল করে duplicate বা পুরনো version জমে আছে কিনা চেক করতে

### 📅 কখন ব্যবহার করব?
- pull করার পর verify করতে যে download সফল হয়েছে
- কোনো container বানানোর আগে কোন image/tag আছে confirm করতে
- Cleanup করার আগে, কোন image মুছে ফেলা দরকার তা বের করতে

### 📝 Syntax

```bash
docker images [OPTIONS] [REPOSITORY[:TAG]]
```

**প্রতিটি অংশের ব্যাখ্যা:**

| অংশ | ব্যাখ্যা |
|-----|----------|
| `-a` | intermediate/build layer images-সহ সব দেখানো |
| `-q` | শুধু IMAGE ID দেখানো |
| `--format` | custom output format |
| `REPOSITORY[:TAG]` | নির্দিষ্ট কোনো image filter করেও দেখা যায় |

### 💡 উদাহরণ

```bash
# সব image দেখা
docker images

# নির্দিষ্ট image-এর সব tag দেখা
docker images nginx

# শুধু IMAGE ID গুলো দেখা (script-এ ব্যবহার করার জন্য কাজে লাগে)
docker images -q

# custom format-এ দেখা
docker images --format "{{.Repository}}:{{.Tag}}  ->  {{.Size}}"
```

### 🖥️ Output ও তার ব্যাখ্যা

```
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
nginx        1.25      9b8c3c0...     2 weeks ago    187MB
node         18-alpine 4f2a1d9...     3 weeks ago    127MB
ubuntu       22.04     a7870f2...     1 month ago    77.9MB
```

| কলাম | ব্যাখ্যা |
|------|----------|
| `REPOSITORY` | image-এর নাম |
| `TAG` | version |
| `IMAGE ID` | প্রতিটা image-এর ইউনিক ID (প্রথম ১২ ক্যারেক্টার দেখায়, পুরো ID দেখতে `--no-trunc` ব্যবহার করা যায়) |
| `CREATED` | image-টা কবে build হয়েছিল (registry-তে, এটা আপনার pull-এর তারিখ না) |
| `SIZE` | image কত জায়গা নিচ্ছে |

### ⚠️ সাধারণ ভুল

> **`<none>` tag দেখে confuse হওয়া** — এগুলোকে "dangling images" বলে, সাধারণত পুরনো build-এর leftover। নিয়মিত `docker image prune` দিয়ে clean করা ভালো।

> **একই IMAGE ID-এর একাধিক নাম দেখে duplicate ভাবা** — আসলে একই image-কে দুইটা tag (নাম) দেওয়া থাকলে এমন দেখায়, এতে অতিরিক্ত space লাগে না।

> **SIZE-কে actual disk usage ভাবা** — layer শেয়ারিং-এর কারণে আসল disk usage এর চেয়ে কম হতে পারে।

### 🔄 Workflow-এ অবস্থান
pull-এর পরের যাচাই ধাপ। এখান থেকে নিশ্চিত হয়ে আমরা পরের ধাপে যাই — সেই image থেকে container বানানো।

---

## ৩. `docker run` — Container তৈরি করে চালানো

### 🔍 কী করে?
এটা একসাথে দুটি কাজ করে:
1. image থেকে একটা নতুন container **তৈরি (create)** করে
2. সেটা **চালু (start)** করে

এটাই Docker-এর সবচেয়ে গুরুত্বপূর্ণ এবং সবচেয়ে বেশি ব্যবহৃত কমান্ড।

### ❓ কেন দরকার?
Image হলো শুধু একটা টেমপ্লেট — তা দিয়ে কিছু execute হয় না, যতক্ষণ না সেটাকে একটা চলমান container-এ পরিণত করা হয়। `run` কমান্ডটাই সেই কাজ করে।

### 📝 Syntax

```bash
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

### 🔧 গুরুত্বপূর্ণ OPTIONS-এর বিস্তারিত

---

#### `-d` / `--detach` — Background-এ চালানো
ব্যাকগ্রাউন্ডে চালানো (terminal লক হবে না)।

---

#### `-p HOST_PORT:CONTAINER_PORT` — Port Mapping

Container-এর ভিতরে একটি Server রান হলে সেটি শুধুমাত্র সেই Container-এর ভিতরেই Access করা যায়। Host Machine (তোমার Laptop বা VPS) থেকে সেটি Access করা যায় না। Browser বা Host Machine থেকে সেই Application Access করতে Container-এর Port-কে Host-এর একটি Port-এর সাথে যুক্ত করতে হয়। এই প্রক্রিয়াকে **Port Mapping** বলা হয়।

```bash
docker run -p 5000:8000 my-app
```

```
Host Machine (Browser)          Container
     Port: 5000      ←——→      Port: 8000
  http://localhost:5000       App runs here
```

| অংশ | অর্থ |
|-----|------|
| `5000` | Host Machine-এর Port (Browser থেকে Access করার Port) |
| `8000` | Container-এর Port (Application যে Port-এ Run করছে) |

---

#### `--name NAME` — Container-এর নাম দেওয়া

Docker-এ Container তৈরি করলে, নাম না দিলে Docker নিজে থেকেই একটা Random নাম দিয়ে দেয়। যেমন: `happy_einstein`, `amazing_turing`, `busy_fermi` — এই Random নামগুলো মনে রাখা কঠিন।

```bash
docker run --name my-backend node
```

> এখন Container-এর নাম হবে: `my-backend`
> পরে সহজে: `docker stop my-backend`, `docker start my-backend`, `docker exec -it my-backend bash`

---

#### `-e KEY=VALUE` — Environment Variable সেট করা

Environment Variable হলো Application-এর Configuration Data, যেগুলো Code-এর বাইরে রাখা হয়।

```bash
# MERN Backend Application-এর জন্য উদাহরণ:
docker run -e PORT=5000 -e NODE_ENV=production my-backend
```

Container-এর ভিতরে গিয়ে `env` চালালে এই Variable গুলো দেখতে পারবে।

> 💡 MERN Application-এ Database URL, JWT Secret, API Key ইত্যাদি সাধারণত Environment Variable হিসেবে রাখা হয়।

---

#### `-v HOST_PATH:CONTAINER_PATH` — Volume Mount (Data Persist করার জন্য)

সাধারণভাবে Container Delete করলে Container-এর ভিতরের সব Data মুছে যায়। কিন্তু বাস্তবে আমরা চাই কিছু Data Container Delete হলেও থেকে যাক।

```bash
docker run -v D:/data:/app/data my-app
```

```
Host Machine           Container
  D:/data    ←——→    /app/data
 (সংরক্ষিত)          (কাজের জায়গা)
```

Container Delete করলেও `D:/data`-তে Data থেকে যাবে — এটাকেই **Data Persistence** বলা হয়।

> ⚠️ একজন MERN Developer হিসেবে MongoDB Container ব্যবহার করার সময় Volume প্রায় **বাধ্যতামূলক**, কারণ Volume ব্যবহার না করলে Container Delete হওয়ার সাথে সাথে Database-এর সমস্ত Data মুছে যাবে।

---

#### `--rm` — Container বন্ধ হলে অটো-ডিলিট

Testing-এর জন্য সুবিধাজনক — container বন্ধ হলেই অটোমেটিক মুছে যাবে।

---

### 💡 উদাহরণ (সব একসাথে)

```bash
# সাধারণভাবে চালানো (foreground-এ, terminal আটকে থাকবে)
docker run nginx

# ব্যাকগ্রাউন্ডে চালানো এবং port map করা
docker run -d -p 8080:80 nginx
# এখন browser-এ localhost:8080 খুললে nginx দেখাবে

# নাম দিয়ে চালানো (পরে এই নামেই handle করা সহজ হবে)
docker run --name my-web -d -p 8080:80 nginx

# Environment variable সহ চালানো
docker run -d -e NODE_ENV=production --name my-app node:18-alpine

# Container বন্ধ হলেই যেন অটো ডিলিট হয়ে যায়
docker run --rm hello-world
```

---

## ৪. `docker run -it` — Interactive Terminal-এ Container চালানো

### 🔍 কী করে?
Container-এর ভিতরে একটা interactive shell session খুলে দেয়, যেন আপনি সরাসরি সেই container-এর ভিতরে বসে কমান্ড চালাচ্ছেন।

### ❓ কেন দরকার?
মাঝে মাঝে container-এর ভিতরে গিয়ে দেখতে হয় — ফাইল আছে কিনা, কোনো command manually চালিয়ে দেখতে হয়, debugging করতে হয়। এর জন্য একটা shell access দরকার, যা সাধারণ `docker run` দেয় না।

### 📅 কখন ব্যবহার করব?
- Container-এর ভিতরের ফাইল সিস্টেম explore করতে
- কোনো নতুন base image (যেমন ubuntu, alpine) test করতে
- Debugging করতে — কেন কোনো অ্যাপ ঠিকমতো চলছে না
- Development/Learning পর্যায়ে

### 📝 Syntax

```bash
docker run -it [OPTIONS] IMAGE [COMMAND]
```

**Flag-এর ব্যাখ্যা:**

| Flag | পূর্ণ রূপ | কাজ |
|------|-----------|-----|
| `-i` | `--interactive` | container-এর stdin (input) খোলা রাখে |
| `-t` | `--tty` | একটা pseudo-terminal (TTY) allocate করে |
| `-it` | দুটো একসাথে | interactive shell-এর জন্য সবসময় একসাথে ব্যবহার করা হয় |

### 💡 উদাহরণ

```bash
# Ubuntu container-এর ভিতরে bash shell ওপেন করা
docker run -it ubuntu bash

# Alpine-ভিত্তিক image-এ sh দিয়ে ঢোকা (alpine-এ bash থাকে না by default)
docker run -it node:18-alpine sh

# একবার ব্যবহারের পর অটো-ডিলিট হবে এমন interactive python session
docker run -it --rm python:3.11 python3

# চলমান container-এর ভিতরে নতুন shell session খোলা (এটা run না, exec — ভিন্ন কনসেপ্ট)
docker exec -it my-web bash
```

### 🖥️ Output ও তার ব্যাখ্যা

```bash
$ docker run -it ubuntu bash
root@e3f1a9b8c7d2:/#
```

- prompt পরিবর্তন হয়ে যায় — `root@<container-id>:/#` — এর মানে আপনি এখন **container-এর ভিতরে** আছেন
- এখানে `ls`, `cat`, `apt install` ইত্যাদি যেকোনো লিনাক্স কমান্ড চালানো যাবে, কিন্তু সেগুলো শুধু এই container-এর ভিতরে প্রভাব ফেলবে
- বের হতে চাইলে `exit` লিখুন বা `Ctrl+D` চাপুন

### ⚠️ সাধারণ ভুল

> **exit করলেই container মুছে যায় ভাবা** — আসলে exit করলে container শুধু বন্ধ (stopped) হয়, ডিলিট হয় না (যদি না `--rm` দেওয়া থাকে)। এটা `docker ps -a`-তে `exited` অবস্থায় দেখা যাবে।

> **`-it` ছাড়া shell চালানোর চেষ্টা করা** — `docker run ubuntu bash` (without `-it`) চালালে shell সাথে সাথেই exit হয়ে যাবে।

> **`run -it` আর `exec -it` গুলিয়ে ফেলা** — `run -it` একটা **নতুন** container বানিয়ে ভিতরে ঢোকে, কিন্তু `exec -it` একটা **ইতিমধ্যে চলমান** container-এর ভিতরে ঢোকে।

> **Container detach করার উপায় না জানা** — container বন্ধ না করে শুধু terminal থেকে বের হতে চাইলে `Ctrl+P` তারপর `Ctrl+Q` চাপতে হয়।

### 🔄 Workflow-এ অবস্থান
এটা মূলত debugging ও exploration-এর ধাপ — development-এ অত্যন্ত গুরুত্বপূর্ণ, কিন্তু production-এ সাধারণত সরাসরি `-it` দিয়ে কিছু চালানো হয় না।

---

## ৫. `docker ps` — চলমান Container দেখা

### 🔍 কী করে?
এই মুহূর্তে যে container গুলো চলছে (running), **শুধু** তাদের তালিকা দেখায়।

### ❓ কেন দরকার?
Container চালানোর পর নিশ্চিত হতে হয় যে সেটা আসলেই চলছে, ঠিকমতো port map হয়েছে কিনা, container-এর নাম/ID কী।

### 📝 Syntax

```bash
docker ps [OPTIONS]
```

| Option | কাজ |
|--------|-----|
| `-q` | শুধু CONTAINER ID দেখায় (script-এ chaining করার জন্য) |
| `--filter` | নির্দিষ্ট শর্তে filter করা |
| `--format` | output-কে custom format-এ দেখানো |

### 💡 উদাহরণ

```bash
# এই মুহূর্তে চলমান সব container দেখা
docker ps

# শুধু ID গুলো দেখা
docker ps -q

# নির্দিষ্ট নামের container আছে কিনা চেক করা
docker ps --filter "name=my-web"
```

### 🖥️ Output ও তার ব্যাখ্যা

```
CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS         PORTS                  NAMES
e3f1a9b8c7d2   nginx     "/docker-entrypoint.…"   5 minutes ago   Up 5 minutes   0.0.0.0:8080->80/tcp   my-web
```

| কলাম | ব্যাখ্যা |
|------|----------|
| `CONTAINER ID` | ইউনিক identifier (short form) |
| `IMAGE` | কোন image থেকে এই container বানানো হয়েছে |
| `COMMAND` | container ভিতরে কোন প্রসেস চলছে |
| `CREATED` | কতক্ষণ আগে তৈরি হয়েছিল |
| `STATUS` | `Up 5 minutes` মানে এটা ৫ মিনিট ধরে চলছে |
| `PORTS` | host port → container port mapping |
| `NAMES` | container-এর নাম |

### ⚠️ সাধারণ ভুল

> **মনে করা `docker ps` সব container দেখায়** — না, এটা শুধু **running** container দেখায়। বন্ধ থাকা container দেখতে `docker ps -a` লাগবে।

> **Empty output দেখে ভয় পাওয়া** — যদি কোনো container না চলে, `docker ps` খালি table দেখাবে, এটা স্বাভাবিক, error না।

> **CONTAINER ID আর IMAGE ID গুলিয়ে ফেলা** — দুটো ভিন্ন জিনিস।

### 🔄 Workflow-এ অবস্থান
`run`-এর পরের যাচাই ধাপ — নিশ্চিত হওয়া যে container আসলেই active আছে।

---

## ৬. `docker ps -a` — সব Container দেখা (চলমান + বন্ধ)

### 🔍 কী করে?
শুধু running না, বরং **সব** container — চলমান, বন্ধ (exited), এমনকি যেগুলো start হয়নি — সবকিছু দেখায়।

### ❓ কেন দরকার?
অনেক সময় একটা container বন্ধ হয়ে গেছে (crash করেছে বা exit হয়ে গেছে), কিন্তু সেটা এখনো ডিস্কে আছে, ডিলিট হয়নি। সেগুলো খুঁজে বের করতে এই কমান্ড লাগে।

### 📝 Syntax

```bash
docker ps -a [OPTIONS]
```

### 💡 উদাহরণ

```bash
# সব container দেখা
docker ps -a

# শুধু exited হওয়া container গুলোর ID দেখা
docker ps -a -q --filter "status=exited"

# শুধু exited status-এর container দেখা
docker ps -a --filter "status=exited"
```

### 🖥️ Output ও তার ব্যাখ্যা

```
CONTAINER ID   IMAGE     COMMAND       CREATED         STATUS                     PORTS     NAMES
e3f1a9b8c7d2   nginx     "/docker-…"   10 min ago      Up 10 minutes              ...       my-web
b2d4e5f6a7c8   ubuntu    "bash"        2 hours ago     Exited (0) 1 hour ago                old-debug
```

**STATUS কলামের অর্থ:**

| STATUS | মানে |
|--------|------|
| `Up X minutes` | চলছে |
| `Exited (0) ...` | স্বাভাবিকভাবে বন্ধ হয়েছে (0 = success) |
| `Exited (1) ...` | কোনো error-এর কারণে বন্ধ হয়েছে |
| `Created` | container তৈরি হয়েছে কিন্তু কখনো start করা হয়নি |

### ⚠️ সাধারণ ভুল

> **Exited container গুলো জমিয়ে রাখা** — এতে ডিস্ক স্পেস নষ্ট হয়। নিয়মিত cleanup করা ভালো অভ্যাস।

> **Exit code উপেক্ষা করা** — `Exited (1)` দেখলে বুঝতে হবে কিছু একটা ভুল হয়েছিল। `docker logs <container>` দিয়ে কারণ খতিয়ে দেখুন।

### 🔄 Workflow-এ অবস্থান
Management ও cleanup-এর আগের অনুসন্ধান ধাপ।

---

## ৭. `docker start` — বন্ধ Container আবার চালু করা

### 🔍 কী করে?
একটি ইতিমধ্যে তৈরি হওয়া কিন্তু বন্ধ থাকা container-কে আবার চালু করে। এটা **নতুন container তৈরি করে না** — বরং পুরনো একটাকেই আবার active করে, তার আগের state (যেমন ফাইল পরিবর্তন) সহ।

### ❓ কেন দরকার?
`docker run` করে একটা container বানিয়ে, তার ভিতরে কিছু কাজ করার পর সেটা বন্ধ হয়ে গেলে:
- `docker run` দিলে — **নতুন** container বানাবে (আগের কাজ থাকবে না)
- `docker start` দিলে — **সেই একই** container আবার চালু হবে (আগের সব data ঠিক থাকবে)

### 📝 Syntax

```bash
docker start [OPTIONS] CONTAINER [CONTAINER...]
```

| Option | কাজ |
|--------|-----|
| `-a` / `--attach` | container-এর output/logs সরাসরি terminal-এ দেখানো |
| `-i` / `--interactive` | STDIN খোলা রাখা |

### 💡 উদাহরণ

```bash
# নাম দিয়ে container চালু করা
docker start my-web

# একসাথে একাধিক container চালু করা
docker start my-web my-db

# বন্ধ থাকা interactive container-এ আবার attach হয়ে শুরু করা
docker start -ai my-debug-container

# সব বন্ধ (exited) container একসাথে চালু করা
docker start $(docker ps -aq -f status=exited)
```

### 🖥️ Output ও তার ব্যাখ্যা

```bash
$ docker start my-web
my-web
```

সফল হলে শুধু container-এর নাম/ID echo হয়ে আসে। এরপর `docker ps` দিয়ে চেক করলে `Up` অবস্থায় দেখাবে।

### ⚠️ সাধারণ ভুল

> **`run` আর `start` গুলিয়ে ফেলা** — মনে রাখার সহজ উপায়:
> - `run` = নতুন container **তৈরি** + চালু
> - `start` = পুরনো (ইতিমধ্যে তৈরি) container শুধু **চালু**

> **`--rm` দিয়ে বানানো container আবার start করার চেষ্টা করা** — যদি কোনো container `--rm` ফ্ল্যাগ দিয়ে বানানো হয়, বন্ধ হলে সেটা স্বয়ংক্রিয়ভাবে ডিলিট হয়ে যায় — তখন আর start করার মতো কিছু থাকে না।

### 🔄 Workflow-এ অবস্থান
stop-start cycle-এর "চালু করা" অংশ।

---

## ৮. `docker stop` — চলমান Container বন্ধ করা

### 🔍 কী করে?
একটা চলমান container-কে **gracefully (সুশৃঙ্খলভাবে)** বন্ধ করে।

**কীভাবে কাজ করে:**
```
SIGTERM পাঠায়  →  ১০ সেকেন্ড অপেক্ষা  →  (বন্ধ না হলে) SIGKILL পাঠায়
(নিজে cleanup করো)                        (জোর করে বন্ধ)
```

### ❓ কেন দরকার?
Container বন্ধ করা দরকার হতে পারে — resource (CPU/RAM) ফ্রি করতে, নতুন version দিয়ে replace করার আগে পুরনোটা বন্ধ করতে — কিন্তু container-টাকে **মুছে ফেলা ছাড়াই**।

### 📝 Syntax

```bash
docker stop [OPTIONS] CONTAINER [CONTAINER...]
```

| Option | কাজ |
|--------|-----|
| `-t` / `--time` | SIGTERM-এর পর কতক্ষণ অপেক্ষা করবে (সেকেন্ডে, default ১০) |

### 💡 উদাহরণ

```bash
# একটা container বন্ধ করা
docker stop my-web

# একাধিক container একসাথে বন্ধ করা
docker stop my-web my-db

# বেশি সময় দিয়ে gracefully বন্ধ করা (ডেটাবেসের জন্য)
docker stop -t 30 my-db

# সব চলমান container একসাথে বন্ধ করা
docker stop $(docker ps -q)
```

### ⚠️ সাধারণ ভুল

> **`stop` আর `rm` গুলিয়ে ফেলা** — `stop` শুধু container **বন্ধ** করে, ডিলিট করে না। Container তখনও ডিস্কে থেকে যায় এবং `docker start` দিয়ে আবার চালু করা যায়।

> **জোর করে kill হওয়ার ব্যাপারটা না বোঝা** — যদি অ্যাপ্লিকেশনটা SIGTERM সঠিকভাবে handle না করে, তাহলে ১০ সেকেন্ড পর জোর করে বন্ধ হবে। Database-এর ক্ষেত্রে `-t` দিয়ে বেশি সময় দেওয়া ভালো।

### 🔄 Workflow-এ অবস্থান
Graceful shutdown-এর ধাপ — `start`-এর বিপরীত জোড়া কমান্ড।

---

## ৯. `docker rm` — Container মুছে ফেলা

### 🔍 কী করে?
একটা বন্ধ থাকা container-কে **স্থায়ীভাবে** মুছে ফেলে — container ও তার metadata ডিস্ক থেকে সরিয়ে দেয়।

> ⚠️ চলমান container সরাসরি মুছা যায় না — আগে `stop` করতে হবে, অথবা `-f` দিয়ে force করতে হবে।

### 📝 Syntax

```bash
docker rm [OPTIONS] CONTAINER [CONTAINER...]
```

| Option | কাজ |
|--------|-----|
| `-f` / `--force` | চলমান container-কেও জোর করে বন্ধ করে মুছে ফেলা |
| `-v` | container-এর সাথে যুক্ত anonymous volume গুলোও মুছে ফেলা |

### 💡 উদাহরণ

```bash
# একটা বন্ধ container মুছে ফেলা
docker rm my-old-test

# একাধিক container একসাথে মুছে ফেলা
docker rm container1 container2 container3

# চলমান container জোর করে বন্ধ করে মুছে ফেলা
docker rm -f my-web

# সব exited container একসাথে মুছে ফেলা (cleanup-এ খুব কাজে লাগে)
docker rm $(docker ps -aq -f status=exited)
```

### 🖥️ Output ও তার ব্যাখ্যা

```bash
$ docker rm my-old-test
my-old-test
```

চলমান container `-f` ছাড়া মুছতে চেষ্টা করলে:
```bash
$ docker rm my-web
Error response from daemon: cannot remove container "my-web": container is running
```

### ⚠️ সাধারণ ভুল

> **চলমান container সরাসরি মুছতে গিয়ে error পাওয়া** — আগে `stop` করে নিতে হবে, অথবা `-f` ব্যবহার করতে হবে।

> **গুরুত্বপূর্ণ data থাকা container ভুলে মুছে ফেলা** — যদি data কোনো volume-এ persist করা না থাকে, container মুছে ফেললে ভিতরের সব data **চিরতরে হারিয়ে যায়**।

> **`-f`-কে সবসময় নিরাপদ ভাবা** — production container-এ ভুলবশত ব্যবহার করলে চলমান সার্ভিস হঠাৎ বন্ধ হয়ে যাবে।

### 🔄 Workflow-এ অবস্থান
Container-level cleanup। সঠিক ক্রম: `stop` → `rm` → (দরকার হলে) `rmi`

---

## ১০. `docker rmi` — Image মুছে ফেলা

### 🔍 কী করে?
লোকাল মেশিন থেকে একটা image **স্থায়ীভাবে** মুছে ফেলে। (`rmi` = remove image)

### ❓ কেন দরকার?
পুরনো বা অব্যবহৃত image গুলো অনেক জায়গা (কয়েকশো MB থেকে কয়েক GB) দখল করে রাখে। এগুলো মুছে ফেলে ডিস্ক স্পেস ফেরত পাওয়া যায়।

### 📝 Syntax

```bash
docker rmi [OPTIONS] IMAGE [IMAGE...]
```

| Option | কাজ |
|--------|-----|
| `-f` / `--force` | জোর করে মুছে ফেলা |

### 💡 উদাহরণ

```bash
# নাম দিয়ে image মুছে ফেলা
docker rmi nginx:1.25

# IMAGE ID দিয়ে মুছে ফেলা
docker rmi 9b8c3c0a1234

# জোর করে মুছে ফেলা
docker rmi -f old-image:latest

# সব dangling (<none> tag) image একসাথে মুছে ফেলা
docker rmi $(docker images -q -f dangling=true)
```

### 🖥️ Output ও তার ব্যাখ্যা

```bash
$ docker rmi nginx:1.25
Untagged: nginx:1.25
Untagged: nginx@sha256:1b87c...
Deleted: sha256:9b8c3c0a1234...
Deleted: sha256:7e6f5d4c3b21...
```

- `Untagged` → image-এর নাম/tag রেফারেন্সটা সরানো হয়েছে
- `Deleted` → আসল image layer গুলো ডিস্ক থেকে মুছে ফেলা হয়েছে

কোনো container এখনও সেই image ব্যবহার করলে:
```
Error response from daemon: conflict: unable to remove repository reference 
"nginx:1.25" (must force) - container e3f1a9b8c7d2 is using its referenced image
```

### ⚠️ সাধারণ ভুল

> **আগে container না মুছেই image মুছতে চাওয়া** — যদি কোনো container সেই image থেকে তৈরি হয়ে থাকে, image মুছা যাবে না। আগে `docker rm` দিয়ে container মুছতে হবে।

> **`rm` (container) আর `rmi` (image) গুলিয়ে ফেলা** — মনে রাখার সহজ উপায়: `rm` = container রিমুভ, `rmi` = **i**mage রিমুভ।

> **সঠিক ক্রম না মানা:**

```
✅ সঠিক ক্রম: container stop → container rm → image rmi
❌ ভুল ক্রম:  image rmi → (error!)
```

### 🔄 Workflow-এ অবস্থান
এটাই পুরো lifecycle-এর **শেষ ধাপ**।

---

## ১১. `docker logs` — Container-এর Output/Log দেখা

### 🔍 কী করে?
Container-এর ভিতরে যে প্রসেসটা চলছে, তার `stdout` ও `stderr` — অর্থাৎ সেই প্রসেস যা কিছু print করেছে — তা বাইরে থেকে দেখায়। **Container-এর ভিতরে না ঢুকেই কাজ করে।**

### ❓ কেন দরকার?
`-d` (detached mode) দিয়ে container চালালে output সরাসরি terminal-এ দেখা যায় না। কিন্তু সেই output (server start হয়েছে কিনা, error এসেছে কিনা) জানা দরকার। `docker logs` সেই output retrieve করে আনে।

### 📝 Syntax

```bash
docker logs [OPTIONS] CONTAINER
```

| Option | কাজ |
|--------|-----|
| `-f` / `--follow` | real-time-এ নতুন log আসার সাথে সাথে দেখানো (যেমন `tail -f`) |
| `--tail N` | শেষের N লাইন দেখানো |
| `-t` / `--timestamps` | প্রতিটা লাইনের সাথে timestamp দেখানো |
| `--since` | নির্দিষ্ট সময়ের পর থেকে log দেখানো (যেমন `--since 10m`) |
| `--until` | নির্দিষ্ট সময় পর্যন্ত log দেখানো |

### 💡 উদাহরণ

```bash
# Container-এর সব log দেখা
docker logs my-app

# শেষের ৫০ লাইন দেখা
docker logs --tail 50 my-app

# Real-time log follow করা
docker logs -f my-app

# Timestamp সহ শেষের ২০ লাইন
docker logs -t --tail 20 my-app

# শেষ ১০ মিনিটের log
docker logs --since 10m my-app
```

### 🖥️ Output ও তার ব্যাখ্যা

```
$ docker logs my-app
> my-app@1.0.0 start
> node server.js

Server listening on port 3000
Connected to MongoDB
GET /api/users 200 12ms
GET /api/users/5 404 3ms
```

- এটা ঠিক সেই output, যা `node server.js` চালালে normal terminal-এ দেখা যেত
- `200`, `404` ইত্যাদি দেখে বোঝা যায় কোন request সফল হয়েছে, কোনটা fail করেছে
- কোনো error stack trace থাকলে এখানেই দেখাবে — **এটাই crash debug করার প্রথম জায়গা**
- `-f` দিয়ে চালালে terminal "follow mode"-এ থাকবে। বের হতে `Ctrl+C` চাপুন (container বন্ধ হবে না)

### ⚠️ সাধারণ ভুল

> **`Ctrl+C` চাপলে container বন্ধ হয়ে যাবে ভেবে ভয় পাওয়া** — `docker logs -f`-এ `Ctrl+C` চাপলে শুধু log দেখা বন্ধ হয়, container background-এ চলতেই থাকে।

> **`--tail` ছাড়া পুরো log দেখতে যাওয়া** — মাসের পর মাস চলা container-এর পুরো log হাজার হাজার লাইনের হতে পারে। সবসময় `--tail` দিয়ে শুরু করা ভালো অভ্যাস।

> **Application নিজে console-এ print না করলে log খালি দেখে confuse হওয়া** — `docker logs` শুধু `stdout/stderr` দেখায়। অ্যাপ্লিকেশন যদি ফাইলে log লেখে, এখানে কিছু দেখা যাবে না।

> **Exited container-এর log দেখা যাবে না ভাবা** — বন্ধ হয়ে যাওয়া container-এর শেষ মুহূর্তের log-ও দেখা যায় (`docker rm` না করা পর্যন্ত)। **তাই crash debug করতে আগে log চেক করুন, তারপর container মুছুন।**

### 🔄 Workflow-এ অবস্থান
Troubleshooting-এর প্রথম ধাপ।

---

## ১২. `docker exec` — চলমান Container-এর ভিতরে কমান্ড চালানো

### 🔍 কী করে?
একটা **ইতিমধ্যে চলমান** container-এর ভিতরে গিয়ে একটা নতুন প্রসেস/কমান্ড চালায় — মূল প্রসেসকে disturb না করেই।

### ❓ কেন দরকার?
শুধু logs দেখে সবসময় সমস্যা বোঝা যায় না — মাঝে মাঝে সরাসরি গিয়ে দেখা দরকার: ফাইল আছে কিনা, environment variable ঠিক আছে কিনা, network connectivity আছে কিনা।

### 📅 কখন ব্যবহার করব?
- Container-এর ভিতরের ফাইল সিস্টেম, config, environment variable দেখতে
- Database container-এ ঢুকে query চালাতে (যেমন mysql shell খোলা)
- Network/connectivity সমস্যা debug করতে (ping, curl চালিয়ে দেখা)
- One-off script বা command চালাতে

### 📝 Syntax

```bash
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
```

| Option | কাজ |
|--------|-----|
| `-it` | interactive shell চালানোর জন্য |
| `-d` | exec করা কমান্ডটাও background-এ চালানো |
| `-u` / `--user` | নির্দিষ্ট user দিয়ে কমান্ড চালানো (যেমন `-u root`) |
| `-w` / `--workdir` | নির্দিষ্ট directory থেকে কমান্ড চালানো |

> ⚠️ **CONTAINER অবশ্যই running থাকতে হবে** — বন্ধ container-এ `exec` কাজ করে না।

### 💡 উদাহরণ

```bash
# Container-এর ভিতরে bash shell খোলা (সবচেয়ে বেশি ব্যবহৃত)
docker exec -it my-app bash

# Alpine-ভিত্তিক image-এ bash না থাকলে sh ব্যবহার করা
docker exec -it my-app sh

# শুধু একটা single command চালানো (shell-এ না ঢুকেই)
docker exec my-app ls /app

# Environment variable চেক করা
docker exec my-app env

# Container-এর ভিতরের একটা ফাইলের content দেখা
docker exec my-app cat /app/config.json

# Database container-এ ঢুকে সরাসরি mysql shell খোলা
docker exec -it my-db mysql -u root -p

# root user হিসেবে ঢোকা (permission সমস্যা debug করতে)
docker exec -it -u root my-app bash

# Network connectivity test করা
docker exec my-app ping -c 3 my-db
```

### 🖥️ Output ও তার ব্যাখ্যা

```bash
$ docker exec -it my-app bash
root@e3f1a9b8c7d2:/app#
```

- prompt পরিবর্তন হয়ে যায় — এটা নতুন container না, বরং চলমান `my-app` container-এরই ভিতরে ঢুকেছে
- `exit` লিখলে শুধু এই exec session বন্ধ হয়, **মূল container চলতেই থাকে**

Single command চালালে:
```bash
$ docker exec my-app ls /app
node_modules
package.json
server.js
```

### ⚠️ সাধারণ ভুল

> **বন্ধ থাকা container-এ exec চালানোর চেষ্টা করা:**
> ```
> Error response from daemon: Container e3f1a9b8c7d2 is not running
> ```
> এই ক্ষেত্রে আগে `docker start` দিয়ে container চালু করতে হবে।

> **`exec -it` আর `run -it` গুলিয়ে ফেলা** — এটাই সবচেয়ে কমন ভুল!
> - `run -it` → সবসময় **নতুন** container তৈরি করে
> - `exec -it` → **চলমান** container-এর ভিতরে ঢোকে
> চলমান কোনো container debug করতে চাইলে সবসময় `exec` ব্যবহার করুন।

> **bash নেই এমন Alpine image-এ bash দিয়ে exec করতে গিয়ে error পাওয়া:**
> ```
> OCI runtime exec failed: exec failed: unable to start container process: 
> exec: "bash": executable file not found
> ```
> এক্ষেত্রে `sh` ব্যবহার করুন।

> **exec দিয়ে করা পরিবর্তন স্থায়ী ভাবা** — exec-এর ভিতরে গিয়ে কিছু install করলে, সেটা শুধু সেই running container-এ থাকবে। Container মুছে গেলে বা নতুন image থেকে container বানালে তা থাকবে না। **স্থায়ী পরিবর্তনের জন্য Dockerfile আপডেট করতে হয়।**

### 🔄 Workflow-এ অবস্থান
Troubleshooting-এর দ্বিতীয় ও গভীরতর ধাপ।

---

## 🔧 একসাথে ব্যবহার: বাস্তব Troubleshooting Flow

ধরুন আপনার `my-app` container ঠিকমতো কাজ করছে না। একজন অভিজ্ঞ developer ঠিক এই ধাপগুলোতে কাজ করে:

```bash
# ধাপ ১: Container আদৌ চলছে কিনা চেক করা
docker ps -a

# ধাপ ২: যদি Exited দেখায়, কেন বন্ধ হলো তা log দিয়ে বোঝা
docker logs --tail 50 my-app

# ধাপ ৩: যদি চলমান থাকে কিন্তু আচরণ অস্বাভাবিক, live log দেখা
docker logs -f my-app

# ধাপ ৪: log থেকে যথেষ্ট তথ্য না পেলে, সরাসরি ভিতরে ঢুকে দেখা
docker exec -it my-app bash

# ধাপ ৫: ভিতরে গিয়ে যাচাই করা
env
cat /app/.env
ls -la /app
ps aux

# ধাপ ৬: প্রয়োজনে network connectivity টেস্ট করা
docker exec my-app curl -v http://my-db:5432
```

---

## 💡 মনে রাখার সহজ নিয়ম

| কমান্ড | মনে রাখার কৌশল |
|--------|----------------|
| `docker pull` | Internet থেকে image **ডাউনলোড** |
| `docker images` | মেশিনে থাকা image-এর **তালিকা** |
| `docker run` | image থেকে container **তৈরি + চালু** |
| `docker ps` | শুধু **চলমান** container দেখা |
| `docker ps -a` | **সব** container দেখা (চলমান + বন্ধ) |
| `docker start` | পুরনো container **চালু** করা |
| `docker stop` | চলমান container **বন্ধ** করা |
| `docker rm` | container **মুছে** ফেলা |
| `docker rmi` | image **মুছে** ফেলা (`i` for image) |
| `docker logs` | বাইরে থেকে log **দেখা** (passive) |
| `docker exec` | ভিতরে গিয়ে কমান্ড **চালানো** (active) |

### 🔑 সবচেয়ে গুরুত্বপূর্ণ পার্থক্যগুলো

```
run vs start:
  run   = নতুন container তৈরি + চালু
  start = পুরনো container শুধু চালু

stop vs rm:
  stop  = container বন্ধ (ডিস্কে থেকে যায়)
  rm    = container মুছে ফেলা (চিরতরে)

rm vs rmi:
  rm    = container রিমুভ
  rmi   = image রিমুভ

run -it vs exec -it:
  run -it   = নতুন container বানিয়ে ভিতরে ঢোকা
  exec -it  = চলমান container-এর ভিতরে ঢোকা

logs vs exec:
  logs = বাইরে থেকে দেখা (passive, read-only)
  exec = ভিতরে গিয়ে করা (active, hands-on)
```

### 📊 Docker Container Lifecycle

```
docker pull ──→ image ডাউনলোড
                    │
                    ↓
docker run  ──→ container তৈরি + চালু
                    │
          ┌─────────┴─────────┐
          │                   │
     docker stop         docker exec
     (বন্ধ করা)          (ভিতরে ঢোকা)
          │
     docker start
     (আবার চালু)
          │
     docker rm
     (মুছে ফেলা)
          │
     docker rmi
     (image মুছা)
```

---

> 💬 **মূল কথা:** `docker logs` আর `docker exec` এই দুটো কমান্ড আয়ত্ত করলে, আপনি আর "container কাজ করছে না" দেখে আটকে থাকবেন না — বরং নিজে থেকেই সমস্যার মূল কারণ খুঁজে বের করতে পারবেন, যেটাই আসল Docker দক্ষতা।
