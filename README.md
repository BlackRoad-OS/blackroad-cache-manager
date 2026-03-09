# 🛣️ blackroad-cache-manager

**Caching layer manager for BlackRoad OS services**

[![npm version](https://img.shields.io/npm/v/@blackroad-os/cache-manager?color=%23FF0066&style=flat-square)](https://www.npmjs.com/package/@blackroad-os/cache-manager)
[![PyPI version](https://img.shields.io/pypi/v/blackroad-cache-manager?color=%23FF0066&style=flat-square)](https://pypi.org/project/blackroad-cache-manager/)
[![License](https://img.shields.io/badge/license-BlackRoad%20Proprietary-7700FF?style=flat-square)](./LICENSE)
[![Node.js](https://img.shields.io/badge/node-%3E%3D18.0.0-0066FF?style=flat-square)](https://nodejs.org/)
[![Python](https://img.shields.io/badge/python-%3E%3D3.10-0066FF?style=flat-square)](https://python.org/)
[![BlackRoad OS](https://img.shields.io/badge/BlackRoad-OS-FF9D00?style=flat-square)](https://blackroad.io)

> High-performance, multi-tier cache orchestration with Redis, in-memory, and edge support — built for the BlackRoad OS routing infrastructure.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Installation](#installation)
  - [npm (Node.js / TypeScript)](#npm-nodejs--typescript)
  - [pip (Python)](#pip-python)
- [Quick Start](#quick-start)
  - [Node.js](#nodejs)
  - [Python](#python)
- [API Reference](#api-reference)
  - [Core Operations](#core-operations)
  - [Hash Operations](#hash-operations)
  - [List Operations](#list-operations)
  - [Set Operations](#set-operations)
  - [Sorted Set Operations](#sorted-set-operations)
  - [Pub/Sub](#pubsub)
  - [Statistics & Monitoring](#statistics--monitoring)
- [Stripe Integration](#stripe-integration)
  - [Caching Stripe Responses](#caching-stripe-responses)
  - [Idempotency Key Storage](#idempotency-key-storage)
  - [Webhook Event Deduplication](#webhook-event-deduplication)
- [Configuration](#configuration)
- [Caching Strategies](#caching-strategies)
- [E2E Testing](#e2e-testing)
- [Architecture](#architecture)
- [Deployment](#deployment)
- [Security](#security)
- [License](#license)

---

## Overview

`blackroad-cache-manager` is the canonical caching layer manager for the BlackRoad OS platform. It provides a unified interface for multi-tier cache orchestration — routing data across in-memory, Redis/Valkey, and Cloudflare KV edge stores — with built-in TTL management, pub/sub, persistence, and Stripe-compatible idempotency handling.

This package is published to both **npm** (TypeScript/JavaScript) and **PyPI** (Python) and is designed to operate at every layer of the BlackRoad routing stack.

---

## Features

| Feature | Description |
|---------|-------------|
| 🔑 **Core Operations** | `get`, `set`, `delete`, `expire`, `ttl`, `keys` |
| 🗂️ **Data Structures** | Strings, Hashes, Lists, Sets, Sorted Sets |
| ⏱️ **TTL Management** | Per-key expiration with background eviction |
| 📡 **Pub/Sub** | Channel-based messaging with subscriber callbacks |
| 🔁 **NX / XX Flags** | Set-if-not-exists and set-if-exists semantics |
| 💾 **Persistence** | Optional SQLite-backed durability |
| 📊 **Metrics** | Hit/miss ratio, memory usage, type breakdown |
| 💳 **Stripe-Ready** | Idempotency key caching and webhook deduplication |
| 🌐 **Edge Support** | Cloudflare KV integration for sub-10ms edge caching |
| 🔒 **Thread-Safe** | Background expiry with thread-safe operations |

---

## Installation

### npm (Node.js / TypeScript)

```bash
npm install @blackroad-os/cache-manager
```

```bash
# Or with Yarn
yarn add @blackroad-os/cache-manager

# Or with pnpm
pnpm add @blackroad-os/cache-manager
```

**Requirements:** Node.js ≥ 18.0.0

### pip (Python)

```bash
pip install blackroad-cache-manager
```

```bash
# Or with Poetry
poetry add blackroad-cache-manager

# Or with uv
uv add blackroad-cache-manager
```

**Requirements:** Python ≥ 3.10

---

## Quick Start

### Node.js

```typescript
import { CacheManager } from '@blackroad-os/cache-manager';

const cache = new CacheManager({
  redis: 'redis://localhost:6379',
  prefix: 'blackroad:',
  defaultTTL: 3600,
  persistence: false,
});

// Set a value with TTL
await cache.set('user:123', { name: 'Alice', plan: 'pro' }, { ttlSeconds: 1800 });

// Get a value
const user = await cache.get('user:123');

// Delete a key
await cache.delete('user:123');

// Pattern delete
await cache.deletePattern('user:*');
```

### Python

```python
from blackroad_cache_manager import CacheManager

cache = CacheManager(enable_persistence=False)

# Set a value with TTL
cache.set("user:123", {"name": "Alice", "plan": "pro"}, ttl_s=1800)

# Get a value
user = cache.get("user:123")

# Delete a key
cache.delete("user:123")

# Get all keys matching a pattern
keys = cache.keys("user:*")
```

---

## API Reference

### Core Operations

| Method | Signature | Description |
|--------|-----------|-------------|
| `set` | `set(key, value, ttl_s?, nx?, xx?)` | Set a key-value pair |
| `get` | `get(key)` | Get value by key; returns `None`/`null` if expired or missing |
| `delete` | `delete(*keys)` | Delete one or more keys; returns count deleted |
| `expire` | `expire(key, ttl_s)` | Set expiration on an existing key |
| `ttl` | `ttl(key)` | Remaining TTL in seconds (`-1` = no expiry, `-2` = not found) |
| `keys` | `keys(pattern?)` | Return keys matching a glob pattern (default: `*`) |
| `flush_db` | `flush_db()` | Clear all keys, TTLs, metadata, and subscriptions |

#### `set` Flags

```python
# NX — Only set if key does NOT exist
cache.set("lock:job-42", 1, ttl_s=30, nx=True)

# XX — Only set if key DOES exist
cache.set("session:abc", token, ttl_s=3600, xx=True)
```

---

### Hash Operations

```python
# Set a hash field
cache.hset("user:123", "email", "alice@example.com")

# Get a hash field
email = cache.hget("user:123", "email")

# Get all fields
profile = cache.hgetall("user:123")  # {"email": "alice@example.com", ...}

# Delete a hash field
cache.hdel("user:123", "email")
```

---

### List Operations

```python
# Push to head / tail
cache.lpush("queue:jobs", "job-1")
cache.rpush("queue:jobs", "job-2")

# Pop from head / tail
job = cache.lpop("queue:jobs")
job = cache.rpop("queue:jobs")

# Get range
items = cache.lrange("queue:jobs", 0, -1)  # All items

# List length
length = cache.llen("queue:jobs")
```

---

### Set Operations

```python
# Add members
cache.sadd("tags:article-1", "python", "cache", "redis")

# Check membership
is_member = cache.sismember("tags:article-1", "python")  # True

# Get all members
tags = cache.smembers("tags:article-1")

# Remove a member
cache.srem("tags:article-1", "redis")

# Set operations
common = cache.sinter("tags:article-1", "tags:article-2")   # Intersection
all_tags = cache.sunion("tags:article-1", "tags:article-2") # Union
```

---

### Sorted Set Operations

```python
# Add with score
cache.zadd("leaderboard", score=9850.5, member="alice")
cache.zadd("leaderboard", score=8720.0, member="bob")

# Get range (ascending by score)
top = cache.zrange("leaderboard", 0, 9)                     # Top 10 members
top_with_scores = cache.zrange("leaderboard", 0, 9, withscores=True)

# Get rank (0-indexed)
rank = cache.zrank("leaderboard", "alice")

# Get score
score = cache.zscore("leaderboard", "alice")                # 9850.5
```

---

### Pub/Sub

```python
def on_event(message):
    print(f"Received: {message}")

# Subscribe
cache.subscribe("events:payments", on_event)

# Publish (returns subscriber count)
delivered = cache.publish("events:payments", {"type": "charge.succeeded"})
```

---

### Statistics & Monitoring

```python
stats = cache.get_stats()
# {
#   "keys": 1042,
#   "memory_bytes": 5242880,
#   "total_accesses": 98721,
#   "hit_rate": 0.943,
#   "type_breakdown": {"string": 780, "hash": 200, "list": 62}
# }
```

---

## Stripe Integration

`blackroad-cache-manager` is designed to work seamlessly with Stripe's API. Cache Stripe API responses, store idempotency keys, and deduplicate webhook events — all with precise TTL control.

### Caching Stripe Responses

Stripe API calls are expensive and rate-limited. Cache responses to reduce latency and stay within rate limits:

```python
import stripe
from blackroad_cache_manager import CacheManager

cache = CacheManager()
stripe.api_key = "sk_live_..."

STRIPE_CACHE_TTL = 300  # 5 minutes

def get_customer(customer_id: str) -> dict:
    cache_key = f"stripe:customer:{customer_id}"
    cached = cache.get(cache_key)
    if cached:
        return cached

    customer = stripe.Customer.retrieve(customer_id)
    cache.set(cache_key, dict(customer), ttl_s=STRIPE_CACHE_TTL)
    return dict(customer)

def get_subscription(subscription_id: str) -> dict:
    cache_key = f"stripe:subscription:{subscription_id}"
    cached = cache.get(cache_key)
    if cached:
        return cached

    subscription = stripe.Subscription.retrieve(subscription_id)
    cache.set(cache_key, dict(subscription), ttl_s=STRIPE_CACHE_TTL)
    return dict(subscription)
```

### Idempotency Key Storage

Prevent duplicate charges by caching Stripe idempotency keys:

```python
import uuid

def create_charge(customer_id: str, amount: int, currency: str = "usd") -> dict:
    idempotency_key = f"charge:{customer_id}:{amount}:{currency}"
    cache_key = f"stripe:idempotency:{idempotency_key}"

    # Check if this charge was already attempted
    existing = cache.get(cache_key)
    if existing:
        return existing  # Return cached result — charge already processed

    charge = stripe.PaymentIntent.create(
        amount=amount,
        currency=currency,
        customer=customer_id,
        idempotency_key=str(uuid.uuid4()),
    )

    # Cache for 24 hours (Stripe's idempotency window)
    cache.set(cache_key, dict(charge), ttl_s=86400)
    return dict(charge)
```

### Webhook Event Deduplication

Stripe may deliver the same webhook event more than once. Deduplicate using the event ID:

```python
def handle_stripe_webhook(payload: bytes, sig_header: str) -> dict:
    event = stripe.Webhook.construct_event(
        payload, sig_header, endpoint_secret="whsec_..."
    )

    dedup_key = f"stripe:webhook:{event['id']}"

    # Deduplicate: ignore if already processed
    if cache.get(dedup_key):
        return {"status": "duplicate", "event_id": event["id"]}

    # Mark as processed (TTL = 48 hours)
    cache.set(dedup_key, True, ttl_s=172800, nx=True)

    # Handle event
    if event["type"] == "charge.succeeded":
        _handle_charge_succeeded(event["data"]["object"])
    elif event["type"] == "customer.subscription.deleted":
        _handle_subscription_cancelled(event["data"]["object"])

    return {"status": "processed", "event_id": event["id"]}
```

---

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `BLACKROAD_REDIS_URL` | `redis://localhost:6379` | Redis/Valkey connection URL |
| `BLACKROAD_CACHE_PREFIX` | `blackroad:` | Global key prefix |
| `BLACKROAD_CACHE_TTL` | `3600` | Default TTL in seconds |
| `BLACKROAD_CACHE_PERSISTENCE` | `false` | Enable SQLite persistence |
| `BLACKROAD_CACHE_DB_PATH` | `~/.blackroad/cache.db` | Path to persistence database |
| `STRIPE_CACHE_TTL` | `300` | Stripe response cache TTL (seconds) |

### Python Configuration

```python
from blackroad_cache_manager import CacheManager

cache = CacheManager(
    db_path="/var/lib/blackroad/cache.db",  # Custom persistence path
    enable_persistence=True,               # Enable SQLite durability
)
```

### Node.js Configuration

```typescript
import { CacheManager } from '@blackroad-os/cache-manager';

const cache = new CacheManager({
  redis: process.env.BLACKROAD_REDIS_URL ?? 'redis://localhost:6379',
  prefix: process.env.BLACKROAD_CACHE_PREFIX ?? 'blackroad:',
  defaultTTL: Number(process.env.BLACKROAD_CACHE_TTL ?? 3600),
  persistence: process.env.BLACKROAD_CACHE_PERSISTENCE === 'true',
});
```

---

## Caching Strategies

### Cache-Aside (Lazy Loading)

```python
def get_user(user_id: str) -> dict:
    cached = cache.get(f"user:{user_id}")
    if cached:
        return cached

    user = db.users.find_by_id(user_id)
    cache.set(f"user:{user_id}", user, ttl_s=1800)
    return user
```

### Write-Through

```python
def update_user(user_id: str, data: dict) -> dict:
    user = db.users.update(user_id, data)
    cache.set(f"user:{user_id}", user, ttl_s=1800)
    return user
```

### Cache Warming

```python
def warm_cache():
    """Pre-populate cache at startup."""
    configs = db.configs.find_all()
    for cfg in configs:
        cache.set(f"config:{cfg['key']}", cfg["value"], ttl_s=86400)
```

### Distributed Lock (via NX flag)

```python
def acquire_lock(resource: str, ttl_s: int = 30) -> bool:
    """Acquire a distributed lock. Returns True if lock was acquired."""
    return cache.set(f"lock:{resource}", 1, ttl_s=ttl_s, nx=True)

def release_lock(resource: str) -> None:
    cache.delete(f"lock:{resource}")
```

---

## E2E Testing

End-to-end tests cover the full cache lifecycle including TTL expiry, Stripe integration, and persistence.

### Running Tests

```bash
# Python
pip install -e ".[dev]"
pytest tests/ -v

# Node.js
npm install
npm test

# With coverage
pytest tests/ --cov=blackroad_cache_manager --cov-report=term-missing
npm run test:coverage
```

### Test Structure

```
tests/
├── unit/
│   ├── test_core_operations.py      # set, get, delete, expire, ttl, keys
│   ├── test_hash_operations.py      # hset, hget, hgetall, hdel
│   ├── test_list_operations.py      # lpush, rpush, lpop, rpop, lrange
│   ├── test_set_operations.py       # sadd, srem, smembers, sinter, sunion
│   ├── test_sorted_set_operations.py # zadd, zrange, zrank, zscore
│   └── test_pubsub.py               # publish, subscribe
├── integration/
│   ├── test_redis_integration.py    # Live Redis/Valkey tests
│   ├── test_stripe_caching.py       # Stripe response caching (mocked)
│   └── test_persistence.py          # SQLite persistence tests
└── e2e/
    ├── test_stripe_e2e.py           # Full Stripe payment flow (test mode)
    ├── test_ttl_expiry.py           # TTL eviction under load
    └── test_concurrent_access.py    # Thread-safety tests
```

### Example E2E Test

```python
import time
import pytest
from blackroad_cache_manager import CacheManager

@pytest.fixture
def cache():
    cm = CacheManager()
    yield cm
    cm.flush_db()

def test_set_get_delete(cache):
    cache.set("hello", "world")
    assert cache.get("hello") == "world"
    cache.delete("hello")
    assert cache.get("hello") is None

def test_ttl_expiry(cache):
    cache.set("temp", "value", ttl_s=1)
    assert cache.get("temp") == "value"
    time.sleep(2)
    assert cache.get("temp") is None  # Expired

def test_nx_flag(cache):
    assert cache.set("once", "first", nx=True) is True
    assert cache.set("once", "second", nx=True) is False
    assert cache.get("once") == "first"

def test_stripe_deduplication(cache):
    event_id = "evt_test_abc123"
    dedup_key = f"stripe:webhook:{event_id}"

    # First delivery — should process
    result = cache.set(dedup_key, True, ttl_s=172800, nx=True)
    assert result is True

    # Second delivery — should be a duplicate
    result = cache.set(dedup_key, True, ttl_s=172800, nx=True)
    assert result is False
```

---

## Architecture

`blackroad-cache-manager` sits at the caching tier of the BlackRoad OS routing stack:

```
┌─────────────────────────────────────────────────────────────┐
│                  BLACKROAD OS ROUTING LAYER                  │
├──────────────────────────────────────────────────────────────┤
│  User Request → Operator → Route to Right Service → Answer  │
└────────────────────────────┬─────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │  CACHE MANAGER  │  ← This package
                    └────────┬────────┘
          ┌──────────────────┼──────────────────┐
          ▼                  ▼                  ▼
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │  In-Memory   │  │ Redis/Valkey │  │ Cloudflare KV│
  │  (L1 Cache)  │  │  (L2 Cache)  │  │  (L3 / Edge) │
  └──────────────┘  └──────────────┘  └──────────────┘
```

### Data Flow

1. **L1 (In-Memory):** Sub-millisecond reads for hot keys. Backed by a Python `dict` or Node.js `Map` with background TTL eviction.
2. **L2 (Redis/Valkey):** Shared cache across all BlackRoad OS nodes. Supports cluster mode with consistent hashing.
3. **L3 (Edge / Cloudflare KV):** Sub-10ms global reads at Cloudflare's edge — used for static configs, feature flags, and public API responses.

### BlackRoad OS Node Integration

| Node | Role | Cache Usage |
|------|------|-------------|
| **octavia** | AI routing decisions | Model metadata, prompt cache |
| **aria** | Agent orchestration | Agent state, task queues |
| **lucidia** | Salesforce / RoadChain | CRM response cache |
| **shellfish** | Public gateway | API response cache, rate limits |

---

## Deployment

### Docker

```dockerfile
FROM python:3.12-slim
WORKDIR /app
RUN pip install blackroad-cache-manager
COPY . .
CMD ["python", "-m", "blackroad_cache_manager"]
```

### Docker Compose (with Redis)

```yaml
version: "3.9"
services:
  cache-manager:
    build: .
    environment:
      - BLACKROAD_REDIS_URL=redis://redis:6379
      - BLACKROAD_CACHE_PREFIX=blackroad:
      - BLACKROAD_CACHE_TTL=3600
    depends_on:
      - redis

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --maxmemory 512mb --maxmemory-policy allkeys-lru
```

### Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blackroad-cache-manager
  namespace: blackroad-os
spec:
  replicas: 3
  selector:
    matchLabels:
      app: blackroad-cache-manager
  template:
    spec:
      containers:
        - name: cache-manager
          image: ghcr.io/blackroad-os/cache-manager:latest
          env:
            - name: BLACKROAD_REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: blackroad-secrets
                  key: redis-url
```

---

## Security

- All cache keys are namespaced with a configurable prefix to prevent collisions.
- Stripe idempotency keys and webhook event IDs are stored with bounded TTLs.
- No Stripe API keys, payment data, or PII are ever stored in the cache.
- Redis connections support TLS via `rediss://` URLs.
- The persistence database is stored at a user-owned path (`~/.blackroad/cache.db` by default) with no world-readable permissions.

---

## License

Copyright © 2026 BlackRoad OS, Inc. All Rights Reserved.

This software is proprietary and confidential. Use is subject to the terms of the [BlackRoad OS Proprietary License](./LICENSE).

---

<p align="center">
  <strong>BlackRoad OS</strong> — Speed at Every Layer<br/>
  <a href="https://blackroad.io">blackroad.io</a> · <a href="https://github.com/BlackRoad-OS">GitHub</a>
</p>
