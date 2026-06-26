# Next.js (TypeScript) Dockerization ও Docker Compose — সম্পূর্ণ গাইড (বাংলা)

## ভূমিকা: Node.js Dockerization থেকে Next.js কোথায় আলাদা?

আগের গাইডে Node.js app Dockerize করেছিলাম। Next.js-ও Node.js-এর উপর চলে — তাহলে কি একই Dockerfile কাজ করবে? **না।** কারণ Next.js-এ একটা extra জিনিস আছে যা Node.js REST API-তে নেই:

```
Node.js API:
  npm install → npm start → ✅ চলছে

Next.js App:
  npm install → npm run build → npm start → ✅ চলছে
  ↑ এই BUILD STEP টাই পার্থক্য তৈরি করে
```

`npm run build` চালালে Next.js পুরো application compile করে `.next/` folder-এ রাখে — JSX → JavaScript, TypeScript → JavaScript, CSS optimize, image optimize, route pre-render ইত্যাদি সব। এই build output-ই production-এ serve হয়।

এই কারণে Next.js-এর জন্য আমাদের **Multi-stage Dockerfile** দরকার — যা Node.js-এর সরল single-stage Dockerfile থেকে আলাদা।

---

## Next.js Dockerization-এর মূল ৩টি চ্যালেঞ্জ

এগুলো না বুঝলে সব Dockerfile লিখেও সমস্যায় পড়বেন:

### চ্যালেঞ্জ ১: Build-time vs Runtime (Image সাইজের সমস্যা)
```
Node.js API এর image:
  node_modules/ (production deps) → ~50-100MB

Next.js App এর image (naive approach):
  node_modules/ (ALL deps + devDeps) → ~500MB+
  .next/ (build output)
  source code

আসলে production-এ কী লাগে:
  .next/standalone/ (optimized server) → ~50MB
  .next/static/ (CSS/JS assets)
  public/ (static files)
```
Solution: **Multi-stage Build** — build করার জন্য আলাদা stage, production-এ run করার জন্য আলাদা lean stage।

### চ্যালেঞ্জ ২: `NEXT_PUBLIC_` Environment Variables
```javascript
// এটা client-side code — browser-এ চলে
const apiUrl = process.env.NEXT_PUBLIC_API_URL;
```
`NEXT_PUBLIC_` prefix দেওয়া variable গুলো **build time-এ** JavaScript bundle-এ bake হয়ে যায় — runtime-এ আর বদলানো যায় না! এটা না বুঝলে `.env` থেকে value নিলেও কাজ করবে না।

### চ্যালেঞ্জ ৩: `output: 'standalone'` না দেওয়া
Next.js-এর একটা শক্তিশালী feature আছে — `standalone` output mode — যা production image-কে আশ্চর্যজনকভাবে ছোট করে। এটা enable না করলে image অনেক বড় হয়।

---

## ধাপ ০: `next.config.ts` — Standalone Mode Enable করা

**সবার আগে এই কাজটা করতে হবে।** বাকি সব এর উপর নির্ভর করে।

```typescript
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  // ✅ এই একটা লাইনই সবচেয়ে গুরুত্বপূর্ণ
  output: 'standalone',
}

export default nextConfig
```

**`output: 'standalone'` কী করে?**

`npm run build` চালালে Next.js এখন `.next/standalone/` folder তৈরি করে, যেখানে:
- Production server চালানোর জন্য **শুধু যা দরকার** তাই থাকে
- `node_modules` থেকে শুধু **actually used** packages নেওয়া হয়
- একটা `server.js` ফাইল তৈরি হয় যা দিয়ে `node server.js` করেই server চলে

```
.next/
├── standalone/           ← Production-এ এটাই লাগে
│   ├── server.js         ← এটা দিয়েই production server চলে
│   ├── node_modules/     ← Minimal, শুধু প্রয়োজনীয় packages
│   └── .next/
│       └── server/
├── static/              ← CSS, JS, fonts (CDN বা public folder-এ serve করা হয়)
└── ...
```

এই কারণে production image-এ পুরো `node_modules` copy করতে হয় না — শুধু `standalone` folder।

---

## প্রজেক্ট স্ট্রাকচার

```
my-nextjs-app/
│
├── src/
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   └── globals.css
│   └── components/
│
├── public/               ← Static assets
│
├── Dockerfile            ← Production multi-stage build
├── Dockerfile.dev        ← Development (hot reload)
├── .dockerignore
├── docker-compose.yml    ← Development
├── docker-compose.prod.yml ← Production
├── .env.local            ← Local development (git-এ না)
├── .env.example          ← Template (git-এ দিন)
├── next.config.ts        ← standalone output enable
├── tsconfig.json
├── package.json
└── package-lock.json
```

---

## ধাপ ১: `.dockerignore` ফাইল

```
# Next.js build output — image-এ নতুন করে build হবে
.next

# Dependencies — image-এ নতুন করে install হবে
node_modules

# Git files
.git
.gitignore

# Environment files — sensitive data
.env
.env.local
.env.*.local

# Docker files নিজেরাই
Dockerfile
Dockerfile.*
docker-compose*.yml

# Development tools
.vscode/
.idea/
*.log
npm-debug.log*

# TypeScript cache
*.tsbuildinfo

# OS files
.DS_Store
Thumbs.db

# Test files
coverage/
__tests__/
*.test.ts
*.spec.ts
cypress/

# README
README.md
```

**Next.js specific গুরুত্বপূর্ণ বিষয়:**
- `.next/` ignore করা **জরুরি** — local build cache image-এ গেলে conflict হয়। Image-এর ভিতরে fresh build হবে।
- `node_modules` ignore — আগের মতোই

---

## ধাপ ২: Production Dockerfile (Multi-stage)

এটাই Next.js Dockerization-এর কেন্দ্রবিন্দু। মনোযোগ দিয়ে পড়ুন:

```dockerfile
# ─────────────────────────────────────────────────────────────────────────────
# Stage 1: deps — শুধু dependencies install করা
# ─────────────────────────────────────────────────────────────────────────────
FROM node:18-alpine AS deps

# libc compatibility — কিছু npm package Alpine-এ চালাতে এটা দরকার
RUN apk add --no-cache libc6-compat

WORKDIR /app

# package.json আগে copy করো (cache optimization — আগের গাইডে শিখেছি)
COPY package*.json ./

# সব dependency install করো (dev সহ, কারণ build-এ TypeScript/ESLint লাগে)
RUN npm ci

# ─────────────────────────────────────────────────────────────────────────────
# Stage 2: builder — Next.js app build করা
# ─────────────────────────────────────────────────────────────────────────────
FROM node:18-alpine AS builder

WORKDIR /app

# Stage 1 থেকে শুধু node_modules নিয়ে আসো (re-install করতে হবে না)
COPY --from=deps /app/node_modules ./node_modules

# Source code copy করো
COPY . .

# ─── NEXT_PUBLIC_ Environment Variables ───────────────────────────────────────
# গুরুত্বপূর্ণ: NEXT_PUBLIC_ variables build time-এ bundle-এ bake হয়।
# তাই এখানেই declare করতে হবে।
# ARG = build argument (docker build --build-arg দিয়ে পাঠানো হয়)
# ENV = এটা দিয়ে ARG-কে environment variable বানানো হয় যা Next.js পড়তে পারে
ARG NEXT_PUBLIC_API_URL
ENV NEXT_PUBLIC_API_URL=${NEXT_PUBLIC_API_URL}

# Telemetry বন্ধ রাখা (optional, privacy)
ENV NEXT_TELEMETRY_DISABLED=1

# ✅ সবচেয়ে গুরুত্বপূর্ণ কমান্ড — Next.js app build করা
# এটা .next/ folder তৈরি করে, standalone output সহ (next.config.ts-এ enable করা)
RUN npm run build

# ─────────────────────────────────────────────────────────────────────────────
# Stage 3: runner — Production server চালানোর জন্য minimal image
# ─────────────────────────────────────────────────────────────────────────────
FROM node:18-alpine AS runner

WORKDIR /app

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

# Security: dedicated non-root user তৈরি করা
# (node:alpine image-এ nextjs/nodejs user তৈরি করছি)
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# ── Builder stage থেকে শুধু দরকারি জিনিসগুলো নিয়ে আসা ──────────────────────

# Public folder (static assets — images, fonts, icons)
COPY --from=builder /app/public ./public

# Standalone folder (minimal production server + minimal node_modules)
# --chown দিয়ে nextjs user-কে owner বানানো হচ্ছে
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./

# Static assets (.next/static — compiled CSS, JS chunks)
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

# Non-root user-এ switch করা
USER nextjs

# Port expose করা
EXPOSE 3000

# Environment variable হিসেবে port সেট করা
# (standalone server এটা পড়ে)
ENV PORT=3000

# ✅ গুরুত্বপূর্ণ: `npm start` না, সরাসরি `node server.js`
# standalone output-এ এই server.js ফাইলটাই production server
CMD ["node", "server.js"]
```

---

## Multi-stage Build — কী হচ্ছে ভিজ্যুয়ালি?

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Stage 1: deps                                                           │
│  node:18-alpine                                                         │
│  + package.json copy                                                    │
│  + npm ci (ALL deps: react, next, typescript, eslint, etc.)             │
│  = node_modules/ (~400MB)                                               │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │ শুধু node_modules/ নিয়ে যায়
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Stage 2: builder                                                        │
│  node:18-alpine                                                         │
│  + node_modules/ (deps থেকে)                                           │
│  + source code copy                                                     │
│  + npm run build → .next/ তৈরি হয়                                      │
│  = পুরো project (~600MB, কিন্তু production-এ এতটুকু লাগে না)          │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │ শুধু ৩টা জিনিস নিয়ে যায়
                               │ → public/
                               │ → .next/standalone/
                               │ → .next/static/
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Stage 3: runner ← এটাই final image যা production-এ চলে               │
│  node:18-alpine (~130MB)                                                │
│  + public/ (~কয়েক MB)                                                  │
│  + .next/standalone/ (~50-80MB, minimal node_modules সহ)               │
│  + .next/static/ (~কয়েক MB)                                            │
│  = Final image: ~180-250MB (আগের ~600MB এর বদলে!)                      │
│                                                                         │
│  CMD: node server.js                                                    │
└─────────────────────────────────────────────────────────────────────────┘
```

প্রতিটা `FROM` মানে নতুন একটা clean slate। আগের stage-এর সব "mess" (dev dependencies, source code, TypeScript files, cache) বাদ পড়ে যায়। শুধু `COPY --from=builder` দিয়ে যা explicitly নেওয়া হয় তাই থাকে।

---

## ধাপ ৩: Environment Variables — সবচেয়ে গুরুত্বপূর্ণ বিষয়

এটা Next.js Dockerization-এর সবচেয়ে confusing অংশ। মনোযোগ দিয়ে পড়ুন:

### দুই ধরনের Environment Variable

```
┌───────────────────────────────────┬──────────────────────────────────────┐
│ NEXT_PUBLIC_* variables           │ Server-side variables                │
├───────────────────────────────────┼──────────────────────────────────────┤
│ Browser-এ চলে                    │ Server-এ চলে (API routes, SSR)       │
│ Build time-এ bundle-এ bake হয়   │ Runtime-এ পড়া হয়                    │
│ Docker build-এর সময় দিতে হয়    │ Docker run-এর সময় দেওয়া যায়       │
│                                   │                                      │
│ NEXT_PUBLIC_API_URL              │ DATABASE_URL                          │
│ NEXT_PUBLIC_SITE_NAME            │ JWT_SECRET                            │
│ NEXT_PUBLIC_STRIPE_KEY           │ MONGODB_URI                           │
└───────────────────────────────────┴──────────────────────────────────────┘
```

### কী সমস্যা হয় না বুঝলে?

```dockerfile
# ❌ ভুল — NEXT_PUBLIC_ variable runtime-এ দিলে কাজ করে না
CMD ["node", "server.js"]
# docker run -e NEXT_PUBLIC_API_URL=http://api.com ...
# এটা কাজ করবে না! Browser bundle-এ undefined থাকবে।

# ✅ সঠিক — Build time-এ ARG দিয়ে দিতে হবে
ARG NEXT_PUBLIC_API_URL
ENV NEXT_PUBLIC_API_URL=${NEXT_PUBLIC_API_URL}
RUN npm run build
# docker build --build-arg NEXT_PUBLIC_API_URL=http://api.com ...
```

### `.env.example` ফাইল (গুরুত্বপূর্ণ)
```env
# .env.example — git-এ দিন

# ── Build time variables (NEXT_PUBLIC_) ──────────────────────────────────────
# এগুলো docker build --build-arg দিয়ে পাঠাতে হবে
NEXT_PUBLIC_API_URL=http://localhost:3001
NEXT_PUBLIC_SITE_NAME=My App

# ── Runtime variables (Server-side only) ─────────────────────────────────────
# এগুলো docker run -e বা compose environment-এ দেওয়া যাবে
DATABASE_URL=mongodb://localhost:27017/mydb
JWT_SECRET=your-jwt-secret-here
NEXTAUTH_SECRET=your-nextauth-secret-here
NEXTAUTH_URL=http://localhost:3000
```

---

## ধাপ ৪: Development Dockerfile (Hot Reload সহ)

Production Dockerfile দিয়ে development করা যায় না — কারণ প্রতিবার code বদলালে পুরো image rebuild করতে হবে। Development-এর জন্য আলাদা Dockerfile:

```dockerfile
# Dockerfile.dev — Development only (hot reload সহ)

FROM node:18-alpine

# libc compatibility
RUN apk add --no-cache libc6-compat

WORKDIR /app

# Dependencies install
COPY package*.json ./
RUN npm ci

# Source code copy করা হচ্ছে না — docker-compose volume mount করবে
# তাই সরাসরি host-এর code use হবে

EXPOSE 3000

# npm run dev = next dev = hot reload সহ development server
CMD ["npm", "run", "dev"]
```

---

## ধাপ ৫: Docker Compose — Development Setup

```yaml
# docker-compose.yml (Development)

services:

  # ── Next.js Frontend ──────────────────────────────────────────────────────
  nextjs:
    build:
      context: .
      dockerfile: Dockerfile.dev       # Development Dockerfile
    container_name: nextjs_dev
    ports:
      - "3000:3000"
    environment:
      # Runtime variables — এখানে দেওয়া যাবে
      - NODE_ENV=development
      - NEXTAUTH_URL=http://localhost:3000
      - NEXTAUTH_SECRET=${NEXTAUTH_SECRET}
      # NEXT_PUBLIC_ variables development-এ .env.local থেকে auto-read হয়
      # কিন্তু container-এ explicitly দিতে হবে
      - NEXT_PUBLIC_API_URL=${NEXT_PUBLIC_API_URL}
      - NEXT_PUBLIC_SITE_NAME=${NEXT_PUBLIC_SITE_NAME}
    volumes:
      # ✅ Hot reload-এর জন্য source code bind mount
      - .:/app
      # ✅ node_modules protect করা (আগের গাইডে শিখেছি)
      - /app/node_modules
      # ✅ .next build cache protect করা (প্রতিবার full rebuild না করতে)
      - /app/.next
    networks:
      - app-network
    restart: unless-stopped

networks:
  app-network:
    driver: bridge
```

**`.env` ফাইল (development):**
```env
# .env
NEXT_PUBLIC_API_URL=http://localhost:3001
NEXT_PUBLIC_SITE_NAME=My Next.js App
NEXTAUTH_SECRET=dev-secret-change-in-production
```

---

## ধাপ ৬: Docker Compose — Next.js + Backend API (Full Stack)

বাস্তবে Next.js একা চলে না — সাথে একটা backend API থাকে। এখন দেখবো কীভাবে Next.js frontend + Node.js/Express backend + MongoDB একসাথে compose করা যায়:

```yaml
# docker-compose.yml (Full Stack Development)

services:

  # ── Service ১: Next.js Frontend ───────────────────────────────────────────
  frontend:
    build:
      context: ./frontend              # frontend folder-এ Dockerfile.dev আছে
      dockerfile: Dockerfile.dev
    container_name: nextjs_frontend
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      # Backend API URL — container network-এ "backend" নামে accessible
      - NEXT_PUBLIC_API_URL=http://localhost:3001   # browser থেকে call
      - NEXT_BACKEND_URL=http://backend:3001        # server-side call (SSR/API routes)
      - NEXTAUTH_URL=http://localhost:3000
      - NEXTAUTH_SECRET=${NEXTAUTH_SECRET}
    volumes:
      - ./frontend:/app
      - /app/node_modules
      - /app/.next
    depends_on:
      - backend
    networks:
      - app-network
    restart: unless-stopped

  # ── Service ২: Node.js/Express Backend API ────────────────────────────────
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
    container_name: express_backend
    ports:
      - "3001:3001"
    environment:
      - NODE_ENV=development
      - PORT=3001
      - MONGODB_URI=mongodb://admin:${MONGO_PASSWORD}@mongodb:27017/myapp?authSource=admin
      - JWT_SECRET=${JWT_SECRET}
    volumes:
      - ./backend:/app
      - /app/node_modules
    depends_on:
      mongodb:
        condition: service_healthy
    networks:
      - app-network
    restart: unless-stopped

  # ── Service ৩: MongoDB ────────────────────────────────────────────────────
  mongodb:
    image: mongo:6
    container_name: mongodb_db
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
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    restart: unless-stopped

# ── Named Volumes ─────────────────────────────────────────────────────────────
volumes:
  mongo-data:

# ── Networks ──────────────────────────────────────────────────────────────────
networks:
  app-network:
    driver: bridge
```

**`.env` ফাইল:**
```env
MONGO_PASSWORD=devpassword123
JWT_SECRET=dev-jwt-secret-change-in-production
NEXTAUTH_SECRET=dev-nextauth-secret
NEXT_PUBLIC_API_URL=http://localhost:3001
```

### গুরুত্বপূর্ণ: NEXT_PUBLIC_API_URL কোনটা?

```
Browser থেকে API call (Client Component):
  NEXT_PUBLIC_API_URL=http://localhost:3001
  ↑ Browser আপনার machine-এ চলে, তাই localhost:3001 সঠিক
  ↑ Docker network-এ "backend" নাম কাজ করবে না (browser Docker network-এ নেই)

Server থেকে API call (Server Component, API Route, SSR):
  NEXT_BACKEND_URL=http://backend:3001
  ↑ Next.js server container Docker network-এ আছে, তাই "backend" hostname কাজ করে
  ↑ এটা NEXT_PUBLIC_ না, তাই সাধারণ runtime env variable
```

```typescript
// Client Component-এ (browser-এ চলে)
const response = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/api/users`);

// Server Component বা API Route-এ (server-এ চলে)
const response = await fetch(`${process.env.NEXT_BACKEND_URL}/api/users`);
```

---

## ধাপ ৭: Production Compose File

```yaml
# docker-compose.prod.yml

services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile           # Production multi-stage Dockerfile
      args:
        # ✅ Build time-এ NEXT_PUBLIC_ variables পাঠানো
        NEXT_PUBLIC_API_URL: ${NEXT_PUBLIC_API_URL}
        NEXT_PUBLIC_SITE_NAME: ${NEXT_PUBLIC_SITE_NAME}
    container_name: nextjs_prod
    ports:
      - "3000:3000"
    environment:
      # Runtime server-side variables
      - NODE_ENV=production
      - NEXTAUTH_URL=${NEXTAUTH_URL}
      - NEXTAUTH_SECRET=${NEXTAUTH_SECRET}
      - NEXT_BACKEND_URL=http://backend:3001
    depends_on:
      - backend
    networks:
      - app-network
    restart: unless-stopped

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: express_prod
    environment:
      - NODE_ENV=production
      - PORT=3001
      - MONGODB_URI=mongodb://admin:${MONGO_PASSWORD}@mongodb:27017/myapp?authSource=admin
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      mongodb:
        condition: service_healthy
    networks:
      - app-network
    restart: unless-stopped

  mongodb:
    image: mongo:6
    container_name: mongodb_prod
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
    restart: unless-stopped

volumes:
  mongo-data:

networks:
  app-network:
    driver: bridge
```

**Production চালানো:**
```bash
docker compose -f docker-compose.prod.yml up -d --build
```

---

## সবচেয়ে কমন ভুলগুলো

### ভুল ১: `next.config.ts`-এ `output: 'standalone'` না দেওয়া
```typescript
// ❌ এই ছাড়া production Dockerfile কাজ করবে না
// .next/standalone/ folder তৈরিই হবে না
const nextConfig: NextConfig = {}

// ✅ সবার আগে এটা করুন
const nextConfig: NextConfig = {
  output: 'standalone',
}
```

### ভুল ২: Production image-এ `npm start` ব্যবহার করা
```dockerfile
# ❌ ভুল — npm start Node.js + Next.js CLI লাগে, যা standalone-এ নেই
CMD ["npm", "start"]

# ✅ সঠিক — standalone server সরাসরি node দিয়ে চলে
CMD ["node", "server.js"]
```

### ভুল ৩: NEXT_PUBLIC_ variable runtime-এ দেওয়ার চেষ্টা করা
```yaml
# ❌ ভুল — এটা কাজ করে না, browser bundle-এ undefined থাকবে
services:
  frontend:
    environment:
      - NEXT_PUBLIC_API_URL=http://api.com  # Runtime-এ দিলে কাজ করে না!

# ✅ সঠিক — build argument হিসেবে দিতে হবে
services:
  frontend:
    build:
      args:
        NEXT_PUBLIC_API_URL: http://api.com  # Build time-এ দিতে হবে
```

### ভুল ৪: `.next` folder `.dockerignore`-এ না রাখা
```
# ❌ .dockerignore-এ .next নেই
# COPY . . করলে local build cache image-এ ঢুকে যায়
# image-এর ভিতরে নতুন build হলে conflict → অদ্ভুত error

# ✅ সবসময় .dockerignore-এ রাখুন
.next
```

### ভুল ৫: Development-এ `.next` volume না দেওয়া
```yaml
# ❌ ভুল — .next bind mount-এ overwrite হয়ে যায়, hot reload ভাঙে
volumes:
  - .:/app
  - /app/node_modules

# ✅ সঠিক — .next আলাদা volume-এ রাখুন
volumes:
  - .:/app
  - /app/node_modules
  - /app/.next          # এটা না দিলে hot reload সঠিকভাবে কাজ নাও করতে পারে
```

### ভুল ৬: Browser vs Server API URL গুলিয়ে ফেলা
```
# ❌ ভুল বোঝাপড়া
NEXT_PUBLIC_API_URL=http://backend:3001   # Browser "backend" চেনে না!

# ✅ সঠিক — দুটো আলাদা variable
NEXT_PUBLIC_API_URL=http://localhost:3001    # Browser-এর জন্য (host machine)
NEXT_BACKEND_URL=http://backend:3001        # Server-এর জন্য (Docker network)
```

### ভুল ৭: Single project-এর জন্য mono-repo structure না ভাবা
```
# ❌ frontend আর backend আলাদা করা ভুলে যাওয়া
my-project/
├── Dockerfile     # কোনটার জন্য?
└── ...

# ✅ সঠিক structure
my-project/
├── frontend/
│   ├── Dockerfile
│   └── Dockerfile.dev
├── backend/
│   ├── Dockerfile
│   └── Dockerfile.dev
└── docker-compose.yml   # root-এ
```

---

## সম্পূর্ণ Development Workflow

```bash
# ── প্রথমবার সেটআপ ───────────────────────────────────────────────────────
# 1. next.config.ts-এ output: 'standalone' যোগ করুন
# 2. .env ফাইল তৈরি করুন
cp .env.example .env
# values বসান

# ── Development ──────────────────────────────────────────────────────────
# সব চালু করুন
docker compose up -d

# Log দেখুন (hot reload কাজ করছে কিনা)
docker compose logs -f frontend

# কোনো সমস্যা হলে ভিতরে ঢোকুন
docker compose exec frontend sh

# ── Code বদলালে ───────────────────────────────────────────────────────────
# .tsx/.ts ফাইল বদলালে → hot reload auto কাজ করবে, কিছু করতে হবে না

# package.json বদলালে (নতুন package) → rebuild
docker compose up -d --build frontend

# next.config.ts বদলালে → restart
docker compose restart frontend

# ── Production Build Test করতে ────────────────────────────────────────────
# Production image locally test করা
docker build -t my-nextjs:prod \
  --build-arg NEXT_PUBLIC_API_URL=http://localhost:3001 \
  .

docker run -p 3000:3000 \
  -e NEXTAUTH_SECRET=test-secret \
  my-nextjs:prod

# ── শেষে ─────────────────────────────────────────────────────────────────
docker compose down
```

---

## Quick Reference

### Dockerfile Stages

| Stage | কাজ | Output |
|---|---|---|
| `deps` | npm ci (all deps) | node_modules/ |
| `builder` | npm run build | .next/standalone/ + .next/static/ |
| `runner` | Production server | Final lean image |

### Environment Variable Rules

| Type | Example | কখন পড়া হয় | কীভাবে দিতে হয় |
|---|---|---|---|
| `NEXT_PUBLIC_*` | `NEXT_PUBLIC_API_URL` | Build time | `docker build --build-arg` |
| Server-side | `DATABASE_URL` | Runtime | `docker run -e` বা compose `environment` |

### Commands

```bash
docker compose up -d                    # Development চালু
docker compose up -d --build           # Rebuild করে চালু
docker compose logs -f frontend        # Live log
docker compose exec frontend sh        # ভিতরে ঢোকা
docker compose down                    # বন্ধ করা

# Production
docker compose -f docker-compose.prod.yml up -d --build
```

---

## মনে রাখার সহজ নিয়ম

```
Node.js Dockerfile:    1 stage (deps install → run)
Next.js Dockerfile:    3 stage (deps → build → run)

NEXT_PUBLIC_ var:  build time-এ bake হয় → ARG দিয়ে পাঠাও
Server-side var:   runtime-এ পড়া হয়  → environment-এ দাও

Browser API URL:   localhost:PORT     (browser Docker-এর বাইরে)
Server API URL:    service-name:PORT  (server Docker network-এ)

Development:  Dockerfile.dev + volume mount = hot reload
Production:   Dockerfile (multi-stage) + standalone = lean image
```

Next.js-এর Dockerization বুঝলে যেকোনো React-based framework (Remix, Vite SSR) এর Dockerization-ও একই logic-এ করতে পারবেন।
