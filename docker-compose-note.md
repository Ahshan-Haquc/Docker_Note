# Docker Compose — সম্পূর্ণ গাইড (বাংলা)

## ভূমিকা: কেন Docker Compose দরকার?

আগের গাইডে আমরা একটা Node.js app Dockerize করতে শিখেছি। কিন্তু বাস্তব দুনিয়ায় কোনো application একা চলে না। একটা সাধারণ web application-এ থাকে:

- **Node.js API** (backend)
- **MongoDB** (database)
- **Nginx** (reverse proxy / static file server)

এই তিনটা service আলাদা container-এ চালানো দরকার। Docker Compose ছাড়া এটা করতে হলে:

```bash
# ❌ Compose ছাড়া — প্রতিটা container আলাদাভাবে চালাতে হয়
docker network create app-network

docker run -d \
  --name mongodb \
  --network app-network \
  -v mongo-data:/data/db \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=secret123 \
  mongo:6

docker run -d \
  --name api \
  --network app-network \
  -p 3000:3000 \
  -e MONGODB_URI=mongodb://admin:secret123@mongodb:27017 \
  -e NODE_ENV=production \
  my-node-app:latest

# এটা nightmare — ৩টা service-এর জন্য ১০+ লাইন, প্রতিদিন এই কমান্ড!
# কেউ নতুন join করলে কী করবে? এই command গুলো কোথাও document করতে হবে।
```

**Docker Compose** এই সমস্যার সমাধান। একটা **`docker-compose.yml`** ফাইলে সব কিছু লিখে রাখলে:

```bash
# ✅ Compose দিয়ে — একটাই কমান্ড
docker compose up -d

# বন্ধ করতে
docker compose down
```

**Docker Compose কী?**
এটা একটা টুল যা দিয়ে **multi-container application** একটা YAML ফাইলে define করে, একটা কমান্ডে সব চালু/বন্ধ করা যায়। এটা YAML ফাইলটা পড়ে সব `docker run`, `docker network create`, `docker volume create` কমান্ড নিজেই চালিয়ে দেয়।

---

## আমরা কী বানাবো?

একটা **Node.js + MongoDB** application, যেখানে থাকবে:

```
┌─────────────────────────────────────────────────────┐
│                Docker Network: app-network           │
│                                                     │
│  ┌──────────────┐         ┌──────────────────────┐  │
│  │  api         │ ──────► │  mongodb             │  │
│  │  Node.js     │         │  MongoDB 6           │  │
│  │  port: 3000  │         │  port: 27017         │  │
│  └──────────────┘         └──────────────────────┘  │
│         │                          │                │
└─────────│──────────────────────────│────────────────┘
          │                          │
     localhost:3000            mongo-data volume
     (browser/Postman)         (data persist)
```

**প্রজেক্ট স্ট্রাকচার:**
```
my-node-app/
│
├── src/
│   └── server.js
│
├── Dockerfile
├── .dockerignore
├── docker-compose.yml          ← আজকের মূল ফাইল
├── docker-compose.prod.yml     ← Production override
├── .env                        ← Secret values (git-এ দেবেন না)
├── .env.example                ← Template (git-এ দেবেন)
├── package.json
└── package-lock.json
```

---

## `docker-compose.yml` — সম্পূর্ণ ফাইল

প্রথমে পুরো ফাইলটা দেখুন, তারপর প্রতিটা অংশ বিস্তারিত শিখবো:

```yaml
# docker-compose.yml

services:

  # ── Service ১: Node.js API ─────────────────────────────────────────────────
  api:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: my_api
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - PORT=3000
      - MONGODB_URI=mongodb://admin:${MONGO_PASSWORD}@mongodb:27017/myapp?authSource=admin
    volumes:
      - .:/app
      - /app/node_modules
    depends_on:
      mongodb:
        condition: service_healthy
    networks:
      - app-network
    restart: unless-stopped

  # ── Service ২: MongoDB ────────────────────────────────────────────────────
  mongodb:
    image: mongo:6
    container_name: my_mongodb
    ports:
      - "27017:27017"
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=${MONGO_PASSWORD}
      - MONGO_INITDB_DATABASE=myapp
    volumes:
      - mongo-data:/data/db
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

# ── Named Volumes ────────────────────────────────────────────────────────────
volumes:
  mongo-data:

# ── Networks ─────────────────────────────────────────────────────────────────
networks:
  app-network:
    driver: bridge
```

এবং **`.env`** ফাইল:
```env
MONGO_PASSWORD=supersecret123
```

---

## YAML ফাইলের প্রতিটি অংশের বিস্তারিত ব্যাখ্যা

### `services` — প্রতিটি Container-এর Definition

```yaml
services:
  api:        ← service-এর নাম (আপনি যা চাইবেন)
    ...
  mongodb:    ← আরেকটা service
    ...
```

`services` হলো Compose ফাইলের **হৃদপিণ্ড**। এখানে প্রতিটা service মানে একটা container। Service-এর নামটাই পরে Docker network-এ **hostname** হিসেবে কাজ করে — অর্থাৎ `api` container থেকে `mongodb` container-কে সরাসরি `mongodb` নামে ডাকা যায়।

---

### `build` vs `image` — Container কোথা থেকে আসবে

**Option ১: `build` — নিজের Dockerfile থেকে image বানানো**
```yaml
api:
  build:
    context: .          # Dockerfile কোথায় আছে (. মানে current directory)
    dockerfile: Dockerfile  # কোন Dockerfile ব্যবহার করবে
```

নিজের লেখা application-এর জন্য `build` ব্যবহার করুন।
এটা লিখলে "docker build ." এটা লেখার প্রয়োজন হয় না, docker compose নিজেই আমার Dockerfile থেকে image তৈরি করে নেবে।

**Option ২: `image` — Docker Hub থেকে ready-made image নেওয়া**
```yaml
mongodb:
  image: mongo:6        # Docker Hub-এর official mongo image, version 6
```

Third-party software (database, cache, proxy) এর জন্য `image` ব্যবহার করুন। এটা automatically `docker pull` করে নেয়।

---

### `container_name` — Container-এর নাম

```yaml
container_name: my_api
```

না দিলে Docker নিজে একটা নাম তৈরি করে (যেমন `my-node-app-api-1`)। দিলে `docker ps`-এ এই নামেই দেখা যাবে এবং `docker logs my_api` এভাবে ব্যবহার করা সহজ হয়।

**সতর্কতা:** `container_name` দিলে একই machine-এ একাধিক instance চালানো যায় না (নাম conflict করে)। Scaling করতে চাইলে এটা বাদ দিন।

---

### `ports` — Host থেকে Container-এ Access

```yaml
ports:
  - "3000:3000"    # "HOST_PORT:CONTAINER_PORT"
  - "8080:80"      # host-এর 8080 → container-এর 80
```

**Format:** `"HOST_PORT:CONTAINER_PORT"`

**গুরুত্বপূর্ণ পার্থক্য:**
```yaml
# ✅ সাধারণ ব্যবহার — host machine থেকে accessible
ports:
  - "27017:27017"

# 🔒 Security-conscious — শুধু অন্য container থেকে accessible, host থেকে না
# (ports section সম্পূর্ণ বাদ দিন, শুধু networks-এ রাখুন)
# Database production-এ এভাবেই রাখা উচিত
```

MongoDB-কে production-এ host-এ expose না করাটাই নিরাপদ — অন্য container গুলো network-এর মাধ্যমে ঠিকই access পাবে।

---

### `environment` — Environment Variables সেট করা

```yaml
environment:
  - NODE_ENV=development              # সরাসরি value
  - PORT=3000
  - MONGODB_URI=mongodb://admin:${MONGO_PASSWORD}@mongodb:27017/myapp?authSource=admin
```

**তিনটা উপায়ে লেখা যায়:**
```yaml
# উপায় ১: List format (সবচেয়ে বেশি ব্যবহৃত)
environment:
  - NODE_ENV=production
  - PORT=3000

# উপায় ২: Map/Object format
environment:
  NODE_ENV: production
  PORT: 3000

# উপায় ৩: .env থেকে নির্দিষ্ট variable নেওয়া (value ছাড়া)
environment:
  - MONGO_PASSWORD    # .env ফাইল থেকে MONGO_PASSWORD এর value নেবে
```

**`${MONGO_PASSWORD}` মানে কী?**
এটা `.env` ফাইল থেকে `MONGO_PASSWORD` variable-এর value নেয়। এটাকে বলে **variable substitution**।

---

### `.env` ফাইল — Secret Values আলাদা রাখা

```env
# .env
MONGO_PASSWORD=supersecret123
```

**কেন আলাদা `.env` ফাইলে রাখা দরকার?**

```yaml
# ❌ ভুল — password সরাসরি compose file-এ
environment:
  - MONGO_INITDB_ROOT_PASSWORD=supersecret123
  # এই ফাইল git-এ push করলে password সবাই দেখতে পাবে!

# ✅ সঠিক — .env ফাইল থেকে নেওয়া
environment:
  - MONGO_INITDB_ROOT_PASSWORD=${MONGO_PASSWORD}
  # .env ফাইল .gitignore-এ থাকবে, git-এ যাবে না
```

**`.env.example`** ফাইলটা git-এ push করুন (actual values ছাড়া):
```env
# .env.example  ← এটা git-এ দিন
MONGO_PASSWORD=your_password_here
```

নতুন developer শুধু `cp .env.example .env` করে নিজের values বসাবে।

---

### `volumes` — Data Persist করা ও Live Code Reload

**দুই ধরনের volume আছে:**

**ধরন ১: Named Volume — Database data persist করা**
```yaml
# Service-এ:
mongodb:
  volumes:
    - mongo-data:/data/db    # named volume : container path

# File-এর শেষে declare করা:
volumes:
  mongo-data:               # Docker manage করে, system-এ কোথাও রাখে
```

এটা MongoDB-র `/data/db` (যেখানে database ফাইল থাকে) কে `mongo-data` নামের একটা Docker-managed volume-এ map করে। Container মুছে ফেললেও data থাকবে।

**ধরন ২: Bind Mount — Development-এ Live Code Reload**
```yaml
api:
  volumes:
    - .:/app                # host current dir → container /app (live sync)
    - /app/node_modules     # container-এর node_modules কে host-এর থেকে আলাদা রাখা
```

এটা Development-এ যাদুকরী: আপনি `server.js` বদলালেই container-এর ভিতরেও সাথে সাথে বদলে যাবে — image rebuild করতে হবে না। `nodemon` দিয়ে চালালে auto-restart হবে।

**`/app/node_modules` লাইনটা কেন?**
```
- .:/app             → host-এর সব কিছু /app-এ map হয়
- /app/node_modules  → কিন্তু /app/node_modules কে এই mapping থেকে বাদ দাও
                       (container-এর নিজের node_modules ব্যবহার করো)
```
এই "anonymous volume trick" না দিলে host-এর `node_modules` (macOS/Windows-এর জন্য compiled) container-এর `node_modules` (Linux-এর জন্য compiled) কে overwrite করবে — crash!

---

### `depends_on` — Service Startup Order

```yaml
api:
  depends_on:
    mongodb:
      condition: service_healthy    # mongodb healthy না হওয়া পর্যন্ত api start হবে না
```

**দুটো option:**
```yaml
# সরল version — mongodb start হলেই api শুরু হবে (healthy হওয়ার আগেই)
depends_on:
  - mongodb

# ভালো version — mongodb সম্পূর্ণ ready হলেই api শুরু হবে
depends_on:
  mongodb:
    condition: service_healthy      # healthcheck pass করতে হবে
```

`service_started` হলে container শুধু শুরু হলেই যথেষ্ট, কিন্তু MongoDB ready হতে ১০-২০ সেকেন্ড লাগে — এই সময়ে Node.js connect করতে গেলে fail করবে। `service_healthy` দিলে MongoDB পুরোপুরি ready হলে তবেই api শুরু হয়।

---

### `healthcheck` — Service কতটা সুস্থ তা চেক করা

```yaml
mongodb:
  healthcheck:
    test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
    interval: 10s      # প্রতি ১০ সেকেন্ডে চেক করবে
    timeout: 5s        # ৫ সেকেন্ডের মধ্যে respond না করলে fail
    retries: 5         # পরপর ৫ বার fail হলে "unhealthy" mark করবে
    start_period: 10s  # শুরুর ১০ সেকেন্ড চেক করবে না (startup time)
```

`test` command-এর exit code `0` হলে healthy, অন্য কিছু হলে unhealthy।

`depends_on: condition: service_healthy` এই healthcheck-এর উপর নির্ভর করে কাজ করে — তাই দুটো একসাথে ব্যবহার করতে হয়।

---

### `networks` — Containers-এর মধ্যে Isolated Communication

```yaml
# প্রতিটা service-এ:
api:
  networks:
    - app-network

mongodb:
  networks:
    - app-network

# File-এর শেষে:
networks:
  app-network:
    driver: bridge
```

**কেন network দরকার?**
- একই network-এ থাকা container গুলো service নাম দিয়ে একে অপরকে ডাকতে পারে
- `api` container থেকে `mongodb:27017` লিখলেই হয় — IP address লাগে না
- আলাদা network-এ থাকা container গুলো একে অপরকে দেখতে পায় না (isolation)

**Default network:** `docker compose up` চালালে Compose নিজেই একটা default network তৈরি করে। Explicit network না দিলেও সব service একই default network-এ থাকে এবং নাম দিয়ে কথা বলতে পারে। কিন্তু explicit network দেওয়া ভালো practice।

---

### `restart` — Container Crash করলে কী করবে

```yaml
restart: unless-stopped    # সবচেয়ে বেশি ব্যবহৃত — manually stop না করা পর্যন্ত restart
```

| Value | আচরণ |
|---|---|
| `no` | কখনো restart করবে না (default) |
| `always` | সবসময় restart করবে (system reboot-এর পরেও) |
| `on-failure` | শুধু error-এ exit হলে restart |
| `unless-stopped` | manually stop করা ছাড়া সবসময় restart |

Development-এ `unless-stopped` বা `no`, Production-এ `unless-stopped` বা `always` ব্যবহার করুন।

---

## Docker Compose-এর Essential Commands

### `docker compose up` — সব কিছু চালু করা

```bash
# Foreground-এ চালানো (সব log একসাথে দেখা যায়, terminal আটকে থাকে)
docker compose up

# ব্যাকগ্রাউন্ডে চালানো (সবচেয়ে বেশি ব্যবহৃত)
docker compose up -d

# Image rebuild করে চালানো (code/Dockerfile বদলালে)
docker compose up -d --build

# শুধু নির্দিষ্ট service চালানো
docker compose up -d mongodb

# Scale করা (একই service-এর একাধিক instance)
docker compose up -d --scale api=3
```

**প্রথমবার `up` চালালে কী হয়:**
```
[+] Running 5/5
 ✔ Network app-network      Created    ← Network তৈরি
 ✔ Volume  mongo-data       Created    ← Volume তৈরি
 ✔ Container my_mongodb     Started    ← MongoDB container চালু
 ✔ Container my_api         Started    ← API container চালু (depends_on অনুযায়ী পরে)
```

---

### `docker compose down` — সব কিছু বন্ধ করা

```bash
# Container বন্ধ + মুছে ফেলা, network মুছে ফেলা (volume রাখে)
docker compose down

# Volume-সহ সব মুছে ফেলা (সতর্কতা: database data হারিয়ে যাবে!)
docker compose down -v

# Image-সহ মুছে ফেলা
docker compose down --rmi all
```

**`stop` vs `down` পার্থক্য:**
```bash
docker compose stop    # Container বন্ধ করে, কিন্তু রেখে দেয় + network রাখে
docker compose down    # Container বন্ধ + মুছে ফেলে + network মুছে ফেলে
```

Development-এ সাধারণত `down` ব্যবহার করি — পরে `up` করলে fresh শুরু হয়।

---

### `docker compose ps` — Compose-এর Container গুলোর Status

```bash
docker compose ps
```

**Output:**
```
NAME          IMAGE              COMMAND                   SERVICE   STATUS         PORTS
my_api        my-node-app-api    "docker-entrypoint.s…"   api       Up 2 minutes   0.0.0.0:3000->3000/tcp
my_mongodb    mongo:6            "docker-entrypoint.s…"   mongodb   Up 2 minutes   0.0.0.0:27017->27017/tcp
```

`docker ps`-এর মতোই, কিন্তু শুধু **এই Compose project-এর** container গুলো দেখায়।

---

### `docker compose logs` — Service-এর Log দেখা

```bash
# সব service-এর log একসাথে দেখা
docker compose logs

# নির্দিষ্ট service-এর log
docker compose logs api

# Real-time follow করা
docker compose logs -f

# নির্দিষ্ট service follow করা (সবচেয়ে বেশি ব্যবহৃত)
docker compose logs -f api

# শেষ ৫০ লাইন দেখা
docker compose logs --tail 50 api
```

**Output:**
```
api      | Server running on port 3000
api      | Environment: development
api      | Connected to MongoDB
mongodb  | MongoDB init process complete
mongodb  | Waiting for connections
```

Service-এর নাম prefix হিসেবে দেখায় — কোন log কোন service-এর তা বোঝা সহজ হয়।

---

### `docker compose exec` — Service-এর Container-এ ঢোকা

```bash
# api container-এ shell খোলা
docker compose exec api sh

# mongodb container-এ mongo shell খোলা
docker compose exec mongodb mongosh -u admin -p

# একটা single command চালানো
docker compose exec api node -e "console.log('hello')"

# Environment variable চেক করা
docker compose exec api env
```

`docker exec -it container_name ...` এর সহজ version — container নামের বদলে service নাম ব্যবহার করা যায়।

---

### `docker compose build` — Image Build/Rebuild করা

```bash
# সব service-এর image build করা
docker compose build

# নির্দিষ্ট service build করা
docker compose build api

# Cache ছাড়া fresh build
docker compose build --no-cache api
```

`docker compose up --build` একসাথে build + up করে — development-এ এটাই বেশি ব্যবহার হয়।

---

### `docker compose restart` — Service Restart করা

```bash
# নির্দিষ্ট service restart
docker compose restart api

# সব service restart
docker compose restart
```

Config বদলালে বা কোনো service stuck হলে restart করতে হয়।

---

## Development vs Production Setup

বাস্তব project-এ development আর production-এ আলাদা config দরকার। এটা করা হয় **multiple compose file** দিয়ে:

**`docker-compose.yml`** (Base — সব environment-এ common):
```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      - NODE_ENV=${NODE_ENV}
      - PORT=3000
      - MONGODB_URI=mongodb://admin:${MONGO_PASSWORD}@mongodb:27017/myapp?authSource=admin
    networks:
      - app-network
    depends_on:
      mongodb:
        condition: service_healthy

  mongodb:
    image: mongo:6
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=${MONGO_PASSWORD}
    volumes:
      - mongo-data:/data/db
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

volumes:
  mongo-data:

networks:
  app-network:
    driver: bridge
```

**`docker-compose.dev.yml`** (Development-এ extra করা):
```yaml
services:
  api:
    ports:
      - "3000:3000"
    volumes:
      - .:/app               # Live code sync
      - /app/node_modules    # node_modules protect
    environment:
      - NODE_ENV=development
    command: ["npm", "run", "dev"]  # nodemon দিয়ে চালানো

  mongodb:
    ports:
      - "27017:27017"        # Dev-এ host থেকে DB access করা যাবে (MongoDB Compass দিয়ে)
```

**`docker-compose.prod.yml`** (Production-এ extra করা):
```yaml
services:
  api:
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
    restart: unless-stopped

  mongodb:
    # Production-এ port expose করা হচ্ছে না — host থেকে access করা যাবে না
    restart: unless-stopped
```

**চালানো:**
```bash
# Development
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## সাধারণ ভুলগুলো

### ভুল ১: `depends_on` দিয়ে মনে করা Service সম্পূর্ণ Ready হলে তারপর Next Service শুরু হবে

```yaml
# ❌ ভুল ধারণা — এটা শুধু "container start" নিশ্চিত করে, "ready" না
depends_on:
  - mongodb

# ✅ সঠিক — healthcheck সহ দেওয়া
depends_on:
  mongodb:
    condition: service_healthy
```

MongoDB container "start" হতে ১ সেকেন্ড লাগে, কিন্তু connection নেওয়ার জন্য "ready" হতে ১০-১৫ সেকেন্ড লাগতে পারে। `service_started` দিলে Node.js আগেই connect করার চেষ্টা করবে, fail করবে।

### ভুল ২: `.env` ফাইল git-এ Push করা

```bash
# ✅ সবসময় .gitignore-এ রাখুন
echo ".env" >> .gitignore

# ✅ .env.example গিটে দিন (actual values ছাড়া)
```

### ভুল ৩: Bind Mount দিয়ে `node_modules` Overwrite করা

```yaml
# ❌ ভুল — host-এর node_modules container-এ যাবে
volumes:
  - .:/app

# ✅ সঠিক — anonymous volume দিয়ে node_modules protect করা
volumes:
  - .:/app
  - /app/node_modules
```

### ভুল ৪: `docker compose down -v` Production-এ চালানো

```bash
# ❌ ভয়ঙ্কর — সব database data মুছে যাবে!
docker compose down -v

# ✅ সাধারণত শুধু এটাই চালান
docker compose down
```

`-v` flag শুধু development-এ fresh start করার জন্য ব্যবহার করুন।

### ভুল ৫: Code বদলে `up -d` (without `--build`) চালানো

```bash
# ❌ ভুল — Dockerfile বা code বদলালেও পুরনো image ব্যবহার হবে
docker compose up -d

# ✅ সঠিক — rebuild করে নতুন image নিয়ে চালানো
docker compose up -d --build
```

### ভুল ৬: MongoDB Connection String-এ localhost ব্যবহার করা

```javascript
// ❌ ভুল — container-এর ভিতরে localhost মানে container নিজে, mongodb না
const uri = "mongodb://localhost:27017";

// ✅ সঠিক — service নাম ব্যবহার করুন (Docker network-এ এটাই hostname)
const uri = "mongodb://admin:password@mongodb:27017/myapp?authSource=admin";
```

---

## সম্পূর্ণ Development Workflow

```bash
# ── প্রথমবার ─────────────────────────────────────────────────────────────
# .env ফাইল তৈরি করুন
cp .env.example .env
# .env ফাইলে আপনার values বসান

# সব চালু করুন
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d

# ── Daily কাজ ────────────────────────────────────────────────────────────
# Status চেক করা
docker compose ps

# Log দেখা
docker compose logs -f api

# কোনো সমস্যা হলে ভিতরে ঢোকা
docker compose exec api sh

# MongoDB-তে সরাসরি query করা
docker compose exec mongodb mongosh -u admin -p

# ── Code বদলালে ───────────────────────────────────────────────────────────
# শুধু server.js বদলালে — bind mount-এর কারণে nodemon auto-restart করবে
# কিছু করতে হবে না!

# package.json বদলালে (নতুন dependency) — rebuild দরকার
docker compose up -d --build api

# ── শেষে ─────────────────────────────────────────────────────────────────
docker compose down
# (volume আছে, পরের দিন up করলেই data ফিরে পাবেন)
```

---

## Quick Reference — Compose Commands

| কমান্ড | কাজ |
|---|---|
| `docker compose up -d` | সব service background-এ চালু |
| `docker compose up -d --build` | Rebuild করে চালু |
| `docker compose down` | সব বন্ধ ও container মুছে ফেলা |
| `docker compose down -v` | Volume-সহ সব মুছে ফেলা |
| `docker compose ps` | সব service-এর status |
| `docker compose logs -f api` | নির্দিষ্ট service-এর live log |
| `docker compose exec api sh` | Service-এর container-এ ঢোকা |
| `docker compose build api` | নির্দিষ্ট service rebuild |
| `docker compose restart api` | নির্দিষ্ট service restart |
| `docker compose stop` | Container বন্ধ (মুছে না) |

---

## মনে রাখার সহজ নিয়ম

```
docker-compose.yml = পুরো application-এর "blueprint"
  services  = প্রতিটা container-এর definition
  volumes   = data কোথায় রাখবো
  networks  = container গুলো কীভাবে কথা বলবে

  build    = নিজের Dockerfile থেকে image বানাও
  image    = Docker Hub থেকে ready-made image নাও
  ports    = host → container port mapping
  env      = runtime configuration
  depends_on = কে আগে, কে পরে চালু হবে

একটা command, সব কিছু:
  docker compose up -d       = সব on
  docker compose down        = সব off
```

Compose শিখলে পরের স্বাভাবিক ধাপ হবে **Docker Networking**, **Multi-stage Builds** এবং প্রথম **CI/CD pipeline** — যেখানে Compose file সরাসরি deployment-এ ব্যবহার হয়।
