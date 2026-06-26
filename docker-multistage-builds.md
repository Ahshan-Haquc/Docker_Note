# Docker Multi-Stage Builds — সংক্ষিপ্ত গাইড (বাংলা)

## সমস্যাটা কী?

একটা application build করতে যা লাগে, আর সেটা run করতে যা লাগে — এই দুটো সম্পূর্ণ আলাদা জিনিস।

```
Build-এ লাগে:                  Run-এ লাগে:
─────────────────               ─────────────────
TypeScript compiler             Compiled JS শুধু
All dev dependencies            Production deps শুধু
Test frameworks                 Server code
Linting tools                   Static assets
Build cache                     Config files
Source maps                     ─────────────────
─────────────────               মোট: ~80MB
মোট: ~600MB
```

Single-stage Dockerfile-এ এই দুটো জিনিস আলাদা করা যায় না — build-এর সব জিনিস final image-এ ঢুকে যায়। Multi-stage build এই সমস্যার সমাধান।

---

## Core Concept — এক কথায়

```dockerfile
# Stage 1: "Builder" — build করার কাজ করো, তারপর ফেলে দাও
FROM node:18-alpine AS builder
# ... build কাজ ...

# Stage 2: "Runner" — builder থেকে শুধু দরকারি জিনিস নাও
FROM node:18-alpine AS runner
COPY --from=builder /app/dist ./dist   # ← এই একটা লাইনই multi-stage-এর সারকথা
```

**`COPY --from=stagename`** — এটাই multi-stage-এর যাদু। এক stage থেকে আরেক stage-এ শুধু নির্দিষ্ট ফাইল নেওয়া যায়, বাকি সব (build tools, source code, cache) বাতিল হয়ে যায়।

---

## Syntax — প্রতিটি অংশ

```dockerfile
# ── Stage define করা ──────────────────────────────────────────────────────
FROM node:18-alpine AS deps
#                   ↑↑↑↑↑
#                   stage-এর নাম (যা খুশি দিন)

FROM node:18-alpine AS builder

FROM node:18-alpine AS runner

# ── এক stage থেকে আরেকটায় ফাইল নেওয়া ───────────────────────────────────
COPY --from=builder /app/dist ./dist
#         ↑↑↑↑↑↑↑   ↑↑↑↑↑↑↑↑  ↑↑↑↑↑
#     stage নাম    source path  dest path

# ── Docker Hub-এর image থেকেও সরাসরি নেওয়া যায় ──────────────────────────
COPY --from=nginx:alpine /etc/nginx/nginx.conf /etc/nginx/nginx.conf
```

**Stage-এর নাম না দিলে** number দিয়ে reference করা যায়:
```dockerfile
FROM node:18-alpine          # Stage 0
FROM node:18-alpine          # Stage 1
COPY --from=0 /app ./app     # Stage 0 থেকে নেওয়া
```
কিন্তু নাম দেওয়া অনেক বেশি readable — সবসময় `AS` দিয়ে নাম দিন।

---

## উদাহরণ ১: Node.js (JavaScript)

```dockerfile
# ── Stage 1: Dependencies ──────────────────────────────────────────────────
FROM node:18-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

# ── Stage 2: Build ────────────────────────────────────────────────────────
FROM node:18-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build           # dist/ folder তৈরি হয়

# ── Stage 3: Production Runner ────────────────────────────────────────────
FROM node:18-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production

# builder থেকে শুধু এটুকুই নিচ্ছি
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./

# Production deps শুধু (dev deps বাদ)
RUN npm ci --only=production && npm cache clean --force

USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

```
Stage 1 (deps):    ~400MB  ─┐
Stage 2 (builder): ~600MB  ─┤ → Final image-এ থাকে না, বাতিল
                             │
Stage 3 (runner):  ~120MB  ◄┘ ← এটাই final image
```

---

## উদাহরণ ২: Node.js (TypeScript)

TypeScript project-এ build step-এ `tsc` (TypeScript compiler) লাগে, কিন্তু production-এ লাগে না:

```dockerfile
# ── Stage 1: Dependencies (dev + prod একসাথে) ────────────────────────────
FROM node:18-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci                  # typescript, ts-node সব install হচ্ছে

# ── Stage 2: TypeScript Compile ───────────────────────────────────────────
FROM node:18-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npx tsc                 # TypeScript → JavaScript (dist/ তৈরি হয়)

# ── Stage 3: Production ───────────────────────────────────────────────────
FROM node:18-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production

# শুধু compiled JS নাও, TypeScript source বাদ
COPY --from=builder /app/dist ./dist
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force
# typescript, @types/*, ts-node — কিছুই নেই এই image-এ

USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

**Final image-এ নেই:**
- `src/` (TypeScript source files)
- `typescript` package (~20MB)
- সব `@types/*` packages
- `ts-node`, `nodemon`, অন্য devDependencies

---

## উদাহরণ ৩: Next.js (আগের গাইড থেকে চেনা)

```dockerfile
FROM node:18-alpine AS deps      # ← Stage 1: npm install
FROM node:18-alpine AS builder   # ← Stage 2: next build
FROM node:18-alpine AS runner    # ← Stage 3: node server.js
```

Next.js-এ `output: 'standalone'` দিলে `.next/standalone/` folder তৈরি হয় — এটাই runner stage-এ নেওয়া হয়।

---

## Specific Stage পর্যন্ত Build করা

```bash
# পুরো Dockerfile build করা (default — শেষ stage পর্যন্ত)
docker build -t my-app .

# শুধু "builder" stage পর্যন্ত build করা (debug/test-এ কাজে লাগে)
docker build --target builder -t my-app:builder .

# "deps" stage পর্যন্ত build করা (dependencies verify করতে)
docker build --target deps -t my-app:deps .
```

`--target` দিয়ে যেকোনো intermediate stage থেমে inspect করা যায় — debugging-এ অনেক কাজে লাগে।

---

## Multi-stage Build-এর ৩টি মূল উপকার

### ১. Image সাইজ কমে
```
Without multi-stage:  ~600MB
With multi-stage:     ~120MB
                      ───────
                      ~80% কম!
```

### ২. Security বাড়ে
Production image-এ build tools নেই → attack surface কম।
যদি container compromise হয়, attacker পাবে শুধু compiled code — source code, compiler, test files কিছুই না।

### ৩. Build Cache আলাদাভাবে কাজ করে
```
deps stage:    package.json না বদলালে → CACHED (npm install skip)
builder stage: source বদলালে → re-run (কিন্তু deps cache থাকে)
runner stage:  শুধু final copy → দ্রুত
```

---

## মনে রাখার নিয়ম

```
Multi-stage = রান্নাঘর vs ডাইনিং টেবিল

রান্নাঘরে (builder stage):
  হাঁড়ি-পাতিল, কাঁচামাল, মশলা, আগুন —
  সব কিছু লাগে রান্নার জন্য

ডাইনিং টেবিলে (runner stage):
  শুধু তৈরি খাবার —
  হাঁড়ি-পাতিল, কাঁচামাল কিছু আসে না

COPY --from=builder = রান্না করা খাবার প্লেটে তুলে দেওয়া
```

**Stage-এর সংখ্যা:**
```
সাধারণ Node.js (JS):        2 stages (builder + runner)
TypeScript বা Next.js:       3 stages (deps + builder + runner)
Complex CI/CD pipeline:      4+ stages (test stage যোগ করা যায়)
```

---

## Quick Reference

```dockerfile
# Stage declare করা
FROM image:tag AS stagename

# অন্য stage থেকে ফাইল নেওয়া
COPY --from=stagename /source/path ./dest/path

# নির্দিষ্ট stage পর্যন্ত build করা
docker build --target stagename -t image-name .
```

```
যা builder-এ থাকে কিন্তু runner-এ আসে না:
  ✗ Source code (TypeScript, JSX)
  ✗ Dev dependencies
  ✗ Build tools (tsc, webpack, babel)
  ✗ Test files
  ✗ node_modules (dev deps)
  ✗ Build cache

যা runner-এ আসে:
  ✓ Compiled output (dist/, .next/standalone/)
  ✓ Production node_modules শুধু
  ✓ Static assets (public/)
  ✓ Config files (যা runtime-এ দরকার)
```
