# 🌐 Docker Networking for MERN Stack Developers

## Docker Networking কী?

Docker Networking হলো এমন একটি ব্যবস্থা যার মাধ্যমে একাধিক Container একে অপরের সাথে Communication করতে পারে।

সহজ ভাষায়,

> Container গুলো আলাদা আলাদা Mini Computer-এর মতো। Networking তাদেরকে একে অপরের সাথে কথা বলার সুযোগ দেয়।

---

## 🤔 Networking কেন দরকার?

একটি MERN Application চিন্তা করো:

```text
Frontend Container (React)
        │
        ▼
Backend Container (Node.js + Express)
        │
        ▼
MongoDB Container
```

এখানে,

* Frontend Backend-এর API Call করবে।
* Backend MongoDB-এর সাথে Connect করবে।

যদি Networking না থাকে,

তাহলে Container গুলো একে অপরকে খুঁজেই পাবে না।

---

# Docker Network Types (যেটা তোমার জানা দরকার)

## 1. Bridge Network ⭐ (Most Important)

এটাই Docker-এর Default Network।

একই Bridge Network-এর Container গুলো একে অপরের সাথে Communicate করতে পারে।

উদাহরণ:

```text
my-network

├── frontend
├── backend
└── mongodb
```

এখন,

backend Container থেকে:

```bash
mongodb:27017
```

দিয়ে MongoDB Access করা যাবে।

IP Address মনে রাখার দরকার নেই।

---

## Network List দেখার Command

```bash
docker network ls
```

Output:

```text
NETWORK ID     NAME       DRIVER

abc123         bridge     bridge
def456         host       host
ghi789         none       null
```

---

## নতুন Network তৈরি করা

```bash
docker network create my-network
```

Output:

```text
my-network
```

---

## Network List Check

```bash
docker network ls
```

Output:

```text
bridge

host

none

my-network
```

---

## Container কে Network এর সাথে Connect করা

### Backend Container

```bash
docker run -d \
--name backend \
--network my-network \
node
```

---

### MongoDB Container

```bash
docker run -d \
--name mongodb \
--network my-network \
mongo
```

এখন Backend Container থেকে:

```bash
mongodb:27017
```

দিয়ে Database Connect করা যাবে।

---

# MERN Example

ধরো তোমার Backend এর `.env` ফাইল:

```env
MONGO_URI=mongodb://mongodb:27017/mydb
```

এখানে,

প্রথম mongodb হলো Container Name।

কারণ Backend এবং MongoDB একই Docker Network এ আছে।

Docker DNS Automatically Container Name Resolve করে।

অর্থাৎ,

```text
mongodb

↓

172.xx.xx.xx (internal IP)

↓

MongoDB Container
```

তোমাকে IP Address লিখতে হবে না।

---

# Host Port vs Container Communication

অনেক Beginner একটা ভুল করে।

তারা ভাবে:

```text
mongodb://localhost:27017
```

এটা Backend Container থেকে কাজ করবে।

❌ ভুল।

কারণ Backend Container-এর ভিতরে localhost মানে Backend Container নিজেই।

MongoDB Container না।

সঠিক হবে:

```text
mongodb://mongodb:27017
```

কারণ,

```text
backend -----> mongodb
```

এখানে mongodb হলো Container Name।

---

# Network Inspect

Network-এর মধ্যে কোন কোন Container আছে দেখতে:

```bash
docker network inspect my-network
```

Output:

```text
Containers:

backend

mongodb

frontend
```

---

# Docker Compose এ Networking

ভালো খবর হলো,

Docker Compose ব্যবহার করলে বেশিরভাগ সময় Network আলাদা করে তৈরি করতে হয় না।

যেমন:

```yaml
services:

  backend:

  mongodb:

  frontend:
```

Docker Compose নিজেই একটি Network তৈরি করে।

এবং,

Backend Container থেকে:

```text
mongodb:27017
```

দিয়ে MongoDB Access করা যায়।

---

# Real MERN Architecture

```text
                    Browser

                       │

                       ▼

            localhost:3000

                       │

                Frontend Container

                       │

                API Request

                       │

                       ▼

              Backend Container

                       │

         mongodb://mongodb:27017

                       │

                       ▼

             MongoDB Container
```

---

# সবচেয়ে গুরুত্বপূর্ণ Commands

Network List:

```bash
docker network ls
```

Network Create:

```bash
docker network create my-network
```

Container Run with Network:

```bash
docker run --network my-network
```

Network Details:

```bash
docker network inspect my-network
```

---

# MERN Developer হিসেবে কী মনে রাখবে?

✅ Container গুলো একই Network এ থাকলে Container Name দিয়েই Communication করা যায়।

✅ localhost শুধুমাত্র নিজের Container-কে নির্দেশ করে।

✅ Backend থেকে MongoDB Connect করতে:

```env
MONGO_URI=mongodb://mongodb:27017/mydb
```

এখানে mongodb হলো MongoDB Container-এর Name।

✅ Production MERN Application-এ সাধারণত Frontend, Backend, MongoDB এবং Redis একই Docker Network এ রাখা হয়।

---

# Final Summary

```text
Create Network

↓

Run Frontend Container

↓

Run Backend Container

↓

Run MongoDB Container

↓

All Containers Join Same Network

↓

Communicate Using Container Names

↓

MERN Application Works Successfully
```
