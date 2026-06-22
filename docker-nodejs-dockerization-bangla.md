# Node.js Application-এর Dockerization — সম্পূর্ণ গাইড (বাংলা)

## ভূমিকা: Dockerization মানে কী?

**Dockerization** মানে হলো আপনার application-কে এমনভাবে প্যাকেজ করা, যাতে সেটা যেকোনো মেশিনে (আপনার laptop, teammate-এর computer, production server) **হুবহু একইভাবে** চলে — কোনো "আমার machine-এ চলছিল কিন্তু server-এ চলছে না" সমস্যা ছাড়াই।

এর জন্য যা লাগে:
- একটা **Dockerfile** — application কীভাবে package হবে তার recipe (এটাই আজকের মূল বিষয়)
- একটা **`.dockerignore`** — কোন ফাইল package-এ নেওয়া যাবে না তার তালিকা
- **`docker build`** — সেই recipe পড়ে একটা image তৈরি করার কমান্ড
- **`docker run`** — সেই image থেকে container চালানোর কমান্ড (আগের গাইডে শিখেছি)

---

## আমরা কী বানাবো?

একটি সাধারণ **Express.js REST API** যেখানে থাকবে:
- `GET /` → Welcome message
- `GET /health` → Health check endpoint

এই ছোট app-টা দিয়ে সম্পূর্ণ Dockerization process শিখবো — কারণ concept বুঝলে যেকোনো বড় Node.js app-এও একই নিয়ম কাজ করবে।

---

## প্রজেক্ট স্ট্রাকচার (শেষে কেমন দেখাবে)

```
my-node-app/
│
├── src/
│   └── server.js          ← আমাদের Express app
│
├── Dockerfile              ← Docker-এর recipe file
├── .dockerignore           ← Docker-এর .gitignore
├── package.json
└── package-lock.json
```

---

## ধাপ ১: Node.js Application তৈরি

### `package.json`
```json
{
  "name": "my-node-app",
  "version": "1.0.0",
  "description": "A simple Express API",
  "main": "src/server.js",
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

### `src/server.js`
```javascript
const express = require('express');
const app = express();

const PORT = process.env.PORT || 3000;

app.use(express.json());

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from Dockerized Node.js!',
    environment: process.env.NODE_ENV || 'development',
    timestamp: new Date().toISOString()
  });
});

app.get('/health', (req, res) => {
  res.json({ status: 'OK' });
});

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
  console.log(`Environment: ${process.env.NODE_ENV || 'development'}`);
});
```

এখন যদি locally চালাতে চান: `npm install` তারপর `npm start` — কিন্তু এখন আমরা এটাকে Docker দিয়ে চালাবো।

---

## ধাপ ২: Dockerfile লেখা

`Dockerfile` হলো Docker-এর কাছে একটা **step-by-step নির্দেশনা** — কীভাবে আমাদের application-এর image বানাতে হবে। এটা একটা plain text file, কোনো extension নেই।

প্রথমে **সম্পূর্ণ Dockerfile** দেখুন, তারপর প্রতিটা লাইন বিস্তারিত ব্যাখ্যা করবো:

```dockerfile
# ─── ধাপ ১: Base Image নির্বাচন ───────────────────────────────────────────────
FROM node:18-alpine

# ─── ধাপ ২: Working Directory সেট করা ─────────────────────────────────────────
WORKDIR /app

# ─── ধাপ ৩: Dependencies আগে Copy করা (Cache optimization) ────────────────────
COPY package*.json ./

# ─── ধাপ ৪: Dependencies Install করা ──────────────────────────────────────────
RUN npm ci --only=production

# ─── ধাপ ৫: বাকি সব Source Code Copy করা ─────────────────────────────────────
COPY . .

# ─── ধাপ ৬: কোন Port expose হবে তা জানানো ────────────────────────────────────
EXPOSE 3000

# ─── ধাপ ৭: Container চালু হলে কোন কমান্ড চলবে ──────────────────────────────
CMD ["node", "src/server.js"]
```

এখন প্রতিটা instruction বিস্তারিত শিখবো।

---

## Dockerfile-এর প্রতিটি Instruction ব্যাখ্যা

### `FROM` — শুরু থেকে না বানিয়ে, একটা ভিত্তির উপর দাঁড়ানো

```dockerfile
FROM node:18-alpine
```

**কী করে?**
প্রতিটা Dockerfile-এর প্রথম instruction (প্রায় সবসময়) এটাই হয়। এটা বলে: "এই image বানানোর জন্য ভিত্তি হিসেবে `node:18-alpine` image ব্যবহার করো।"

**কেন দরকার?**
আমাদের application run করতে হলে Node.js runtime দরকার। সেটা নিজে থেকে install করার বদলে, Docker Hub-এ `node` নামে একটা official image আছে যেখানে Node.js আগে থেকেই installed। আমরা সেটাকে **base/parent image** হিসেবে ব্যবহার করি এবং তার উপরে আমাদের application বসাই।

**`node:18-alpine` মানে কী?**
- `node` → Docker Hub-এর official Node.js image
- `18` → Node.js version 18 (LTS)
- `alpine` → Alpine Linux-ভিত্তিক, যা অত্যন্ত ছোট (মাত্র ~5MB OS), তাই image সাইজ অনেক কম হয়

**বিভিন্ন variant-এর পার্থক্য:**
```dockerfile
FROM node:18           # Full Debian Linux — বড় (~950MB), সব tool আছে
FROM node:18-slim      # Debian, minimal packages — মাঝারি (~240MB)
FROM node:18-alpine    # Alpine Linux — সবচেয়ে ছোট (~130MB), production-এ সেরা
```

**Production Best Practice:** সবসময় নির্দিষ্ট version pin করুন। `FROM node:latest` বা `FROM node:18` ব্যবহার না করে `FROM node:18-alpine` বা আরও specific `FROM node:18.19-alpine3.18` ব্যবহার করা ভালো, যাতে image predictable থাকে।

---

### `WORKDIR` — Container-এর ভিতরে কাজের জায়গা ঠিক করা

```dockerfile
WORKDIR /app
```

**কী করে?**
Container-এর ভিতরে `/app` নামের একটা directory তৈরি করে (যদি না থাকে) এবং বলে: "এখন থেকে Dockerfile-এর বাকি সব command এই directory-তে বসে চলবে।"

**কেন দরকার?**
`WORKDIR` না দিলে সব ফাইল container-এর root directory (`/`)-তে copy হবে — যেখানে `bin`, `etc`, `usr` এই system ফোল্ডারগুলোও আছে। এটা বিশৃঙ্খল এবং বিপজ্জনক। `/app` (অথবা `/usr/src/app`) ব্যবহার করা একটা convention।

**কী হয় এর ফলে?**
এরপরের সব `COPY`, `RUN`, `CMD` instruction `/app`-এর ভিতরে কাজ করবে। `exec -it container bash` দিয়ে container-এ ঢুকলেও সরাসরি `/app`-এ থাকবেন।

---

### `COPY` — আমাদের ফাইল Image-এ নেওয়া

```dockerfile
COPY package*.json ./
```

**কী করে?**
Host machine (আপনার কম্পিউটার) থেকে file/folder নিয়ে image-এ রাখে।

**Syntax:**
```
COPY <source> <destination>
```
- `<source>` → আপনার machine-এ কোথা থেকে নেবে (`.dockerignore`-এ বাদ দেওয়া ছাড়া)
- `<destination>` → container-এ কোথায় রাখবে (`./` মানে current `WORKDIR`, অর্থাৎ `/app`)

**`package*.json` মানে কী?**
`*` wildcard — এটা `package.json` এবং `package-lock.json` দুটোই match করে। `package-lock.json` দিলে `npm ci` exact dependency versions install করতে পারে।

**কেন আগে শুধু `package.json` copy করি, তারপর বাকি সব?**

এটা Docker-এর সবচেয়ে গুরুত্বপূর্ণ **cache optimization কৌশল** — নিচে বিস্তারিত ব্যাখ্যা আছে।

---

### `RUN` — Image Build করার সময় Command চালানো

```dockerfile
RUN npm ci --only=production
```

**কী করে?**
Image build করার সময় (container চালানোর সময় না) একটা command চালায় এবং সেই command-এর ফলাফল image-এ সংরক্ষিত হয়।

**`npm ci` vs `npm install` — পার্থক্য কী?**

| বিষয় | `npm install` | `npm ci` |
|---|---|---|
| Lock file | Optional | Required (package-lock.json চাই) |
| Version | Flexible (range allow) | Exact lock file version |
| Speed | ধীর (resolve করে) | দ্রুত (সরাসরি install) |
| node_modules | Incremental update | পুরো ফোল্ডার মুছে নতুন করে |
| ব্যবহার | Development | Production/CI |

`npm ci` production-এ ব্যবহার করা উচিত কারণ এটা reproducible (সবসময় একই version install করে)।

**`--only=production` কেন?**
`devDependencies` (যেমন `nodemon`, `jest`, testing tools) production-এ দরকার নেই। এই flag দিলে শুধু `dependencies` install হয়, `devDependencies` বাদ যায় — ফলে image সাইজ কমে।

**`RUN` vs `CMD` vs `ENTRYPOINT`:**
- `RUN` → build time-এ চলে, layer তৈরি করে (যেমন `apt-get install`, `npm ci`)
- `CMD` → container run করার সময় চলে (default command)
- `ENTRYPOINT` → container-এর main process (advanced, আজকে বিস্তারিত না)

---

### বাকি Source Code Copy করা

```dockerfile
COPY . .
```

**কী করে?**
Current directory (`.`) -র সব ফাইল image-এর current `WORKDIR` (`.`, মানে `/app`)-এ copy করে।

**কেন এটা `package.json` copy-এর পরে করি?**

এটাই Docker build-এর সবচেয়ে গুরুত্বপূর্ণ কৌশল — **Layer Caching**:

```
COPY package*.json ./    ← Layer A: শুধু package.json পরিবর্তন হলে এটা re-run হবে
RUN npm ci               ← Layer B: Layer A পরিবর্তন হলেই শুধু এটা re-run হবে
COPY . .                 ← Layer C: Source code পরিবর্তন হলে এটা re-run হবে
```

**বাস্তব পার্থক্য:**

```
❌ ভুল অর্ডার (সব এক সাথে copy, তারপর install):

COPY . .
RUN npm ci               ← প্রতিবার কোনো .js ফাইল বদলালেও npm ci চলবে (ধীর!)

✅ সঠিক অর্ডার:

COPY package*.json ./
RUN npm ci               ← শুধু package.json বদলালে এটা re-run হবে
COPY . .                 ← .js ফাইল বদলালে শুধু এই layer নতুন হবে, npm ci cache থেকেই
```

প্রতিদিনের development-এ আপনি `server.js` বদলাবেন অনেকবার, কিন্তু `package.json` কদাচিৎ বদলাবেন। সঠিক অর্ডারে dependencies layer cache হয়ে থাকে, ফলে প্রতিবার `npm ci` চলে না — build অনেক দ্রুত হয়।

---

### `EXPOSE` — Container কোন Port-এ কথা বলবে তা জানানো

```dockerfile
EXPOSE 3000
```

**কী করে?**
Documentation হিসেবে বলে: "এই container port 3000-এ listen করে।"

**গুরুত্বপূর্ণ:** এটা কিন্তু আসলে কোনো port খুলে দেয় না! এটা শুধু একটা **metadata/hint** — যে এই image ব্যবহার করবে তাকে জানানো যে কোন port-এ connect করতে হবে।

আসল port mapping হয় `docker run -p` দিয়ে:
```bash
docker run -p 8080:3000 my-node-app
# host-এর 8080 → container-এর 3000 (EXPOSE করা port)
```

**তাহলে EXPOSE কেন লিখবো?**
- Document হিসেবে কাজ করে — অন্য developer বুঝতে পারে কোন port ব্যবহার করতে হবে
- Docker Compose এবং Kubernetes কিছু ক্ষেত্রে এই info ব্যবহার করে
- `docker inspect` দিয়ে দেখা যায়

---

### `CMD` — Container চালু হলে কী করবে

```dockerfile
CMD ["node", "src/server.js"]
```

**কী করে?**
Container start হলে এই command চালায়। এটাই container-এর **main/primary process**।

**Syntax — দুটো ফর্ম আছে:**
```dockerfile
# ✅ Exec form (recommended) — JSON array format
CMD ["node", "src/server.js"]

# ❌ Shell form (avoid করুন)
CMD node src/server.js
```

**কেন Exec form (JSON array) ব্যবহার করবেন?**

Shell form ব্যবহার করলে command একটা shell (`/bin/sh -c`) দিয়ে চলে, ফলে:
- Signal handling (যেমন `SIGTERM` থেকে `docker stop`) ঠিকমতো কাজ করে না
- Alpine-এ `sh` আলাদা আচরণ করতে পারে

Exec form সরাসরি process চালায়, shell ছাড়া — তাই `docker stop` এর `SIGTERM` সরাসরি Node.js process-এ পৌঁছায়, graceful shutdown হয়।

**`CMD` vs `ENTRYPOINT`:**
```dockerfile
# শুধু CMD দিলে — docker run-এ argument দিয়ে override করা যায়
CMD ["node", "src/server.js"]
# docker run my-app sh   ← এটা CMD override করে sh চালাবে

# ENTRYPOINT দিলে — সহজে override হয় না, container-এর fixed behavior
ENTRYPOINT ["node", "src/server.js"]
```

Beginner হিসেবে `CMD` দিয়ে শুরু করুন, এটাই বেশি flexible।

---

## ধাপ ৩: `.dockerignore` ফাইল

`.gitignore`-এর মতোই, `.dockerignore` বলে দেয় `COPY . .` করার সময় কোন ফাইল/ফোল্ডার **বাদ** দিতে হবে।

```
# Dependencies - এগুলো image-এ install হবে, copy করতে হবে না
node_modules

# Git files - image-এর দরকার নেই
.git
.gitignore

# Log files
*.log
npm-debug.log*

# Environment files - sensitive data, image-এ রাখা উচিত না
.env
.env.local
.env.*.local

# Development-only files
.dockerignore
Dockerfile
Dockerfile.*
docker-compose*.yml

# OS generated files
.DS_Store
Thumbs.db

# Test files - production image-এ লাগবে না
__tests__
*.test.js
*.spec.js
coverage/

# Editor config
.vscode/
.idea/
```

**কেন `.dockerignore` এত গুরুত্বপূর্ণ?**

```
node_modules ছাড়া:
  COPY . .  →  image-এ আপনার code copy হয়, তারপর
  RUN npm ci  →  clean install হয় ✅

node_modules সহ:
  COPY . .  →  আপনার local node_modules (হয়তো ভুল OS-এর জন্য compile হওয়া)
                image-এ copy হয়ে যায় 💥
  RUN npm ci  →  এই duplicate mess-এর উপর আবার install ❌
```

`node_modules` locally হয়তো macOS-এর জন্য compile হয়েছে, কিন্তু container Linux-এ চলে — তাই local `node_modules` সরাসরি copy করলে native module crash করতে পারে। সর্বদা `node_modules` `.dockerignore`-এ রাখুন।

---

## ধাপ ৪: `docker build` — Image তৈরি করা

এখন Dockerfile আর `.dockerignore` তৈরি হয়ে গেছে। Image বানানোর সময়:

### Syntax
```
docker build [OPTIONS] PATH
```
- `PATH` → Dockerfile কোথায় আছে (সাধারণত current directory `.`)
- `-t` / `--tag` → image-কে একটা নাম:tag দেওয়া
- `-f` → নির্দিষ্ট Dockerfile path দেওয়া (default: `./Dockerfile`)
- `--no-cache` → cache ব্যবহার না করে fresh build করা

### উদাহরণ
```bash
# সবচেয়ে সাধারণ — current directory থেকে build করা
docker build -t my-node-app .

# Version tag সহ (production-এ সবসময় এটা করুন)
docker build -t my-node-app:1.0.0 .

# একসাথে দুটো tag দেওয়া
docker build -t my-node-app:1.0.0 -t my-node-app:latest .

# Cache bypass করে সম্পূর্ণ fresh build
docker build --no-cache -t my-node-app .
```

### Build Output ও তার ব্যাখ্যা

```
$ docker build -t my-node-app .

[+] Building 28.3s (9/9) FINISHED
 => [internal] load build definition from Dockerfile                    0.0s
 => => transferring dockerfile: 512B                                    0.0s
 => [internal] load .dockerignore                                       0.0s
 => [internal] load metadata for docker.io/library/node:18-alpine      1.2s
 => [1/5] FROM docker.io/library/node:18-alpine@sha256:...             8.5s
 => [2/5] WORKDIR /app                                                  0.0s
 => [3/5] COPY package*.json ./                                         0.0s
 => [4/5] RUN npm ci --only=production                                 18.1s
 => [5/5] COPY . .                                                      0.0s
 => exporting to image                                                   0.4s
 => => naming to docker.io/library/my-node-app:latest                  0.0s
```

- **`[1/5]`** থেকে **`[5/5]`** → Dockerfile-এর প্রতিটা instruction একটা করে layer তৈরি করে
- `FROM`-এর layer সবচেয়ে বেশি সময় নেয় (base image download) — পরের build-এ এটা cache থেকে আসে
- `RUN npm ci` সবচেয়ে বেশি সময় নেয় — কিন্তু `package.json` না বদললে পরের build-এ এটাও cache থেকে আসে

**দ্বিতীয় build (কোনো .js ফাইল বদলে):**
```
 => CACHED [1/5] FROM docker.io/library/node:18-alpine          0.0s  ← Cache!
 => CACHED [2/5] WORKDIR /app                                   0.0s  ← Cache!
 => CACHED [3/5] COPY package*.json ./                          0.0s  ← Cache!
 => CACHED [4/5] RUN npm ci --only=production                   0.0s  ← Cache! (package.json বদলায়নি)
 => [5/5] COPY . .                                              0.0s  ← নতুন (code বদলেছে)
```
মাত্র ১-২ সেকেন্ড! Cache সঠিকভাবে ব্যবহার হচ্ছে।

---

## ধাপ ৫: Container চালানো ও Test করা

```bash
# Basic run
docker run -p 3000:3000 my-node-app

# ব্যাকগ্রাউন্ডে run করা (সাধারণত এটাই করা হয়)
docker run -d -p 3000:3000 --name my-app my-node-app

# Environment variable সহ চালানো
docker run -d \
  -p 3000:3000 \
  --name my-app \
  -e NODE_ENV=production \
  -e PORT=3000 \
  my-node-app

# Container চলছে কিনা চেক করা
docker ps

# Log দেখা (server start হয়েছে কিনা নিশ্চিত হতে)
docker logs my-app

# Test করা
curl http://localhost:3000
curl http://localhost:3000/health
```

**Expected output:**
```bash
$ curl http://localhost:3000
{
  "message": "Hello from Dockerized Node.js!",
  "environment": "production",
  "timestamp": "2024-01-15T10:30:00.000Z"
}

$ curl http://localhost:3000/health
{"status": "OK"}
```

---

## ধাপ ৬: Production-Ready Dockerfile

উপরের Dockerfile কাজ করে, কিন্তু production-এ কিছু extra সতর্কতা দরকার। এখন সবচেয়ে ভালো version দেখুন:

```dockerfile
# ─────────────────────────────────────────────────────────────────────────────
# Stage: Production
# ─────────────────────────────────────────────────────────────────────────────
FROM node:18-alpine

# ── ১. Security: root user-এ না চালিয়ে একটা dedicated user তৈরি করা ─────────
#    Root হিসেবে container চালানো Linux-এ root হিসেবে সব চালানোর মতোই বিপজ্জনক।
#    node:alpine image-এ "node" নামে একটা non-root user আগে থেকেই তৈরি আছে।
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser

# ── ২. Working Directory সেট করা ──────────────────────────────────────────────
WORKDIR /app

# ── ৩. Dependencies আগে copy করা (cache optimization) ─────────────────────────
COPY package*.json ./

# ── ৪. Production dependencies install করা ────────────────────────────────────
RUN npm ci --only=production && npm cache clean --force

# ── ৫. Source code copy করা ───────────────────────────────────────────────────
COPY --chown=appuser:appgroup . .

# ── ৬. Non-root user-এ switch করা ─────────────────────────────────────────────
USER appuser

# ── ৭. Port expose করা ────────────────────────────────────────────────────────
EXPOSE 3000

# ── ৮. Application চালানো ─────────────────────────────────────────────────────
CMD ["node", "src/server.js"]
```

**নতুন কী যোগ হলো?**

**Non-root user (`appuser`):**
```dockerfile
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
# ...
COPY --chown=appuser:appgroup . .
USER appuser
```
- Default-এ Docker container `root` হিসেবে চলে
- যদি কোনো vulnerability exploit হয়, attacker container-এ root access পাবে
- Dedicated `appuser` দিয়ে চালালে ক্ষতির পরিধি সীমিত থাকে
- `--chown` দিয়ে copy করা ফাইলের owner সেই user-কে করা হচ্ছে

**npm cache clean:**
```dockerfile
RUN npm ci --only=production && npm cache clean --force
```
- `npm ci` চালানোর পর npm-এর download cache ডিস্কে থাকে
- Production image-এ এই cache আর দরকার নেই, তাই মুছে image সাইজ কমানো হয়

---

## সবচেয়ে কমন ভুলগুলো

### ভুল ১: `node_modules` `.dockerignore`-এ না রাখা
```
# ❌ .dockerignore-এ node_modules নেই
# ফলে COPY . . local node_modules copy করে ফেলে
# → Image বড় হয়, wrong platform-এর binary থাকে, npm ci মেসি হয়

# ✅ সবসময় .dockerignore-এ রাখুন
node_modules
```

### ভুল ২: Layer cache ভুল অর্ডারে করা
```dockerfile
# ❌ ভুল - প্রতিবার .js বদলালে npm install চলবে (অনেক ধীর build)
COPY . .
RUN npm install

# ✅ সঠিক - package.json না বদলালে npm install cache থেকে আসবে
COPY package*.json ./
RUN npm ci --only=production
COPY . .
```

### ভুল ৩: `.env` ফাইল Image-এ ঢুকিয়ে দেওয়া
```
# ❌ .dockerignore-এ .env নেই
# COPY . . দিলে .env image-এ যাবে
# Image push করলে Docker Hub-এ secret চলে যাবে!

# ✅ সবসময় .dockerignore-এ রাখুন
.env
.env.*

# Secret গুলো runtime-এ দিন:
docker run -e DATABASE_URL="..." -e JWT_SECRET="..." my-app
# অথবা docker-compose এর environment section থেকে
```

### ভুল ৪: Shell form CMD ব্যবহার করা
```dockerfile
# ❌ Shell form - SIGTERM ঠিকমতো handle হবে না, graceful shutdown কাজ করবে না
CMD node src/server.js

# ✅ Exec form - সবসময় এটা ব্যবহার করুন
CMD ["node", "src/server.js"]
```

### ভুল ৫: `latest` tag দিয়ে image push করা
```bash
# ❌ latest মানে কোন version? কেউ জানে না
docker build -t my-app:latest .

# ✅ নির্দিষ্ট version tag দিন
docker build -t my-app:1.2.3 -t my-app:latest .
# দুটোই দেওয়া যায় — specific version traceability-র জন্য, latest convenience-এর জন্য
```

### ভুল ৬: `npm install` production-এ ব্যবহার করা
```dockerfile
# ❌ npm install - package-lock.json ignore করে, flexible version install করে
RUN npm install

# ✅ npm ci - exact versions, faster, CI/CD-র জন্য তৈরি
RUN npm ci --only=production
```

---

## সম্পূর্ণ Workflow — একনজরে

```bash
# ── প্রজেক্ট রেডি হলে ─────────────────────────────────────────────────────

# ১. Image Build করা
docker build -t my-node-app:1.0.0 .

# ২. Build সফল হয়েছে কিনা চেক করা
docker images my-node-app

# ৩. Container চালানো
docker run -d \
  --name my-app \
  -p 3000:3000 \
  -e NODE_ENV=production \
  my-node-app:1.0.0

# ৪. চলছে কিনা চেক করা
docker ps

# ৫. Log দেখা (server ঠিকমতো start হয়েছে কিনা)
docker logs my-app

# ৬. Application test করা
curl http://localhost:3000
curl http://localhost:3000/health

# ── কোনো সমস্যা হলে ──────────────────────────────────────────────────────
docker logs my-app                    # কী error হলো
docker exec -it my-app sh             # ভিতরে ঢুকে investigate করা

# ── Code বদলালে ──────────────────────────────────────────────────────────
docker stop my-app                    # পুরনো container বন্ধ
docker rm my-app                      # পুরনো container মুছে ফেলা
docker build -t my-node-app:1.0.1 .   # নতুন image build
docker run -d --name my-app -p 3000:3000 my-node-app:1.0.1

# ── Cleanup ───────────────────────────────────────────────────────────────
docker stop my-app
docker rm my-app
docker rmi my-node-app:1.0.0
```

---

## Quick Reference — Dockerfile Instructions

| Instruction | কখন চলে | কাজ |
|---|---|---|
| `FROM` | Build time | Base image নির্বাচন |
| `WORKDIR` | Build time | Working directory সেট |
| `COPY` | Build time | Host → Image ফাইল copy |
| `RUN` | Build time | Command চালিয়ে layer তৈরি |
| `EXPOSE` | Metadata | কোন port এর document |
| `ENV` | Build+Run time | Environment variable সেট |
| `CMD` | Run time | Default container command |
| `USER` | Build+Run time | কোন user-এ চলবে |

---

## মনে রাখার সহজ নিয়ম

```
Dockerfile = রান্নার রেসিপি
  FROM      = কোন চুলা/পাত্র ব্যবহার করবো (base OS+runtime)
  WORKDIR   = কোন ঘরে রান্না হবে
  COPY      = উপকরণ নিয়ে আসা
  RUN       = রান্না করা (build time)
  CMD       = পরিবেশন করা (run time)
```

এই ৫টা instruction দিয়ে যেকোনো Node.js application Dockerize করা যায়। বাকি সব (`USER`, `ENV`, `HEALTHCHECK`, `ARG`, `VOLUME`) হলো additional best practices — যা ধীরে ধীরে শিখলেই হবে।
