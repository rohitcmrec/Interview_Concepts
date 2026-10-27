# Redis Caching: The QA Perspective

A combined interview-prep guide covering how Redis caching is tested, and how cached data stays fresh when the database changes.

---

## 1. How to explain it in 30 seconds

> "Redis is an in-memory cache that sits between the application and the database. On a read, the app checks Redis first. If the data is there (cache hit), it returns immediately. If not (cache miss), it fetches from the DB, stores it in Redis with a TTL, and returns it. As a QA, I verify that caching actually improves performance, that the data stays correct and fresh, and that the system behaves safely when the cache fails."

---

## 2. What a QA actually tests

### 2.1 Functional testing
- **Cache miss flow:** First request goes to DB, and the value gets stored in Redis. Verify the key exists with the right format (e.g., `product:42`).
- **Cache hit flow:** Second request is served from Redis, and the DB is not queried (check DB logs or query counters).
- **Response consistency:** Cached response must be identical to the DB response.
- **TTL:** Key expires after the configured time, and the next request triggers a fresh DB fetch.

### 2.2 Data consistency (the most important area)
- **Update scenario:** Update a record via API, then read it. You should get the new value, not stale data.
- **Delete scenario:** Delete a record, then read it. It should not be served from cache.
- **Invalidation:** Verify the cache key is deleted or updated after a write.
- **Multi-user scenario:** User A updates, User B reads, and B should see the latest data.

### 2.3 Performance testing
- Compare response time with cache vs. without cache (e.g., 80 ms vs. 5 ms).
- Measure **cache hit ratio** (hits / total requests). Target is usually 80-95%+ for read-heavy data.
- Run load tests (JMeter, k6, Gatling) and confirm DB load and CPU drop.
- Check behavior under spikes.

### 2.4 Negative and resilience testing
- **Redis down:** The app should fall back to the DB, not crash or return 500s.
- **Redis slow or timeout:** Verify timeouts are configured and requests don't hang.
- **Cache stampede:** A popular key expires and many requests hit the DB at once. Verify only one (or a few) go through.
- **Memory full:** Verify the eviction policy works (e.g., LRU) and the app still functions.
- **Invalid or corrupted cache data:** The app should handle it and refetch from DB.
- **Cache penetration:** Requests for non-existent IDs always miss. Verify the app handles this (e.g., caching null results briefly).

### 2.5 Security
- Redis should not be exposed publicly and should require authentication.
- Sensitive data (passwords, tokens, PII) should not be cached in plain form, or should have short TTLs.
- Check that one user's cached data is never returned to another user (user-specific keys).

---

## 3. How the latest value reaches the user after a DB update

**The key question:** If data is updated in the DB but the key is still in Redis, the app keeps serving the old value until the TTL expires. Redis has no idea the DB changed. **Keeping the cache fresh is the application's job.**

### 3.1 Strategies

**1. Invalidate on write (most common)**

When the app updates the DB, it also deletes the cache key:

```python
def update_product(product_id, data):
    db.update_product(product_id, data)   # 1. update DB
    r.delete(f"product:{product_id}")     # 2. delete cache key
```

The next read is a cache miss, so it fetches the fresh value from the DB and re-caches it. Deleting is preferred over updating the cache because it's simpler and avoids race conditions.

**2. Write-through (update cache on write)**

The app writes the new value to both DB and Redis, so the cache is always current.

**3. Event-based invalidation**

Useful when many services or jobs update the same data. The updater publishes an event (Kafka, Redis Pub/Sub, or DB change-data-capture such as Debezium), and a consumer deletes or refreshes the affected keys.

**4. Short TTL (safety net)**

Even with invalidation, a TTL ensures any missed invalidation self-heals eventually. For data that changes often, use a shorter TTL.

**5. Versioned keys**

Include a version in the key (`product:42:v7`). When data changes, the version increments, so the old key is never read again and expires on its own.

### 3.2 The tricky scenario: DB updated outside the app

If someone updates the DB directly (a SQL script, a batch job, an admin tool, another service) and that path doesn't clear the cache, Redis stays stale until the TTL ends. This is a classic real-world bug and a great thing to mention in an interview.

**Fixes:** route all writes through one service, use CDC or events, or accept a bounded staleness window with a short TTL.

### 3.3 A subtle race condition

Even with invalidation, this can happen:
1. Request A reads the DB (old value) on a cache miss.
2. Request B updates the DB and deletes the key.
3. Request A then writes the old value into Redis.

Now the cache is stale again until the TTL. This is why TTL is always kept as a backstop.

### 3.4 Test cases for freshness

| Test | Steps | Expected |
|---|---|---|
| Update via API | GET (cached) → PUT/update → GET | New value returned; key was deleted or updated |
| Check key | After update, run `GET product:42` in redis-cli | Key missing (or holds the new value) |
| Direct DB update | Change the DB with SQL, then GET the API | Old value until TTL expires (reveals a design gap, so raise it) |
| TTL expiry | Wait for TTL, then GET | Fresh value from DB |
| Delete | Delete record via API, then GET | 404, not the cached record |
| Concurrent update and read | Fire parallel reads and updates | No lasting stale data |

---

## 4. Tools and commands you can mention

**redis-cli commands** (these show hands-on experience):

```
KEYS product:*        # list keys (avoid in production, use SCAN)
GET product:42        # view cached value
TTL product:42        # remaining time to live (-2 = expired/missing)
DEL product:42        # manually clear to test a cache miss
FLUSHALL              # clear everything (test env only)
INFO stats            # keyspace_hits and keyspace_misses
MONITOR               # watch live commands
```

**Other tools:**
- **Postman / REST Assured:** compare response times for first vs. second call
- **JMeter / k6:** load testing
- **RedisInsight:** GUI to inspect keys and TTLs
- **DB query logs / APM (Grafana, New Relic, Datadog):** confirm the DB isn't being hit

---

## 5. Sample test scenario to narrate

> "I tested a product API backed by Redis. First call: I confirmed with `redis-cli` that the key didn't exist, the response took ~120 ms, and the DB query log showed a hit. After the call, `GET product:42` returned the cached JSON and `TTL` showed about 300 seconds. Second call took ~8 ms with no DB query. Then I updated the product through the API and verified the key was invalidated, so the next read returned the new price. I also stopped Redis to confirm the API fell back to the DB without errors, and ran a JMeter load test showing the hit ratio around 90% and DB CPU dropping significantly."

---

## 6. Likely interview questions and short answers

**Q: What is a cache hit vs. miss?**
Hit: data found in Redis. Miss: not found, so fetch from DB and store in cache.

**Q: How do you verify data is actually coming from cache?**
Check `redis-cli` for the key, compare response times, and check that DB logs show no query on the second call.

**Q: If the DB is updated but the key is still in Redis, how does the app get the latest value?**
It doesn't automatically. Redis doesn't know the DB changed. The application deletes or updates the key on every write, so the next read refetches from the DB. TTL acts as a safety net for any missed invalidation.

**Q: What if the DB is updated directly, bypassing the app?**
The cache stays stale until the TTL expires. Fixes include routing all writes through one service, using CDC or events to invalidate keys, or setting a short TTL.

**Q: What's the biggest risk with caching?**
Stale data. I test update and delete flows to make sure invalidation works and TTL is appropriate.

**Q: What happens if Redis goes down?**
A well-designed app falls back to the DB. I test this by stopping Redis and checking for graceful degradation, with slower responses but no failures.

**Q: What is cache stampede?**
Many requests hitting the DB at once when a hot key expires. Mitigated with locking, jittered TTLs, or early refresh. I'd test with concurrent requests.

**Q: What is a TTL and why does it matter?**
Time to live, the expiry for a key. It bounds how stale data can get and prevents memory from filling up.

**Q: How do you measure cache effectiveness?**
Hit ratio from `INFO stats` (`keyspace_hits / (keyspace_hits + keyspace_misses)`), plus latency and DB load comparisons.

**Q: Redis vs. a database?**
Redis is in-memory, so it's very fast but more expensive and limited in size. It's used for temporary, frequently read data, while the DB is the durable source of truth.

---

## 7. Ready-made interview answer on staleness

> "Redis doesn't know when the DB changes, so staleness is handled by the application. Typically we delete the cache key whenever the record is updated, so the next read refetches from the DB. TTL acts as a safety net. As a QA, I test the update and delete flows to confirm the key is invalidated, and I also test cases where the DB is updated outside the API, since those can leave stale data until the TTL expires."

---

## 8. One honest tip

Interviewers often follow up with "what did you actually do?" Only claim what you can back up. If you haven't used every tool above, run through the basic flow once on a local Redis (`docker run -p 6379:6379 redis`) using `redis-cli`: check a key, view its TTL, delete it, update the source data, and watch hits and misses in `INFO stats`. Fifteen minutes of hands-on practice will make your answers sound natural and credible.
