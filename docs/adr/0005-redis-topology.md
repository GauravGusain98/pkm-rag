# ADR-0005: Two Redis instances, one for the job queue and one for the cache

- **Status:** accepted
- **Date:** 2026-11-20
- **Phase:** 0

## Context
Redis has two jobs in this system, and they need opposite settings:

| | Job queue (Celery broker, from Phase 4) | Cache (query embeddings and answers, from Phases 7 and 11) |
|---|---|---|
| If a key is lost | A document stays in `queued` forever. Nothing crashes and nothing logs an error. | A small slowdown: the value is recomputed. |
| Must it survive a crash? | Yes | No |
| When memory is full | Must never drop keys | Should drop keys freely |

Two Redis mechanisms are involved, and they're independent:
- **Persistence:** whether data survives a restart. RDB takes periodic snapshots (by default `3600 1 / 300 100 / 60 10000`), so a job queued 4 minutes before a crash can be lost. AOF logs every write and fsyncs about once a second.
- **Eviction:** what happens when `maxmemory` is reached. `maxmemory-policy` picks which keys get deleted, or `noeviction` rejects writes instead. This is separate from **expiry** (TTL), which deletes keys whose time has run out under every policy.

`maxmemory`, the eviction policy and persistence are settings for the **whole instance**. Separate logical databases (`/0`, `/1`) share them, so they can't give the queue and the cache different settings.

## Options considered
1. **One instance with `volatile-lru` and AOF.** Only keys with a TTL can be evicted, so queue keys are safe *if every cache key has a TTL*.
   - Pros: one container, simple.
   - Cons: depends on a rule that's easy to break. A cache key written without a TTL can never be evicted and slowly fills memory. Keys that libraries write with a TTL, such as Celery's result keys (`result_expires`), *can* be evicted. The cache is also persisted to disk for no benefit.
2. **One instance with `noeviction` and AOF.** This is what Celery's docs recommend for a broker.
   - Pros: the queue never loses data.
   - Cons: when the cache fills memory, **every** write fails, including enqueueing jobs, so uploads return a 500 or leave documents stuck in `queued`. A cache problem breaks uploads (a noisy neighbor).
3. **Two instances: `redis-queue` (AOF + `noeviction`) and `redis-cache` (`allkeys-lru`, no AOF, no RDB).**
   - Pros: each gets the right settings. The cache filling up is a non-event. There's no TTL rule to remember. The queue and cache are separate failure domains.
   - Cons: one more container (a few MB of RAM), two connection URLs, and one more thing to monitor.

## Decision
Option 3. The queue and the cache have opposite requirements, so they run as separate instances:
- **`redis-queue`** (host port 6380): `--appendonly yes`, `--maxmemory 256mb`, `--maxmemory-policy noeviction`, and a named volume. RDB is also left on (the default), which gives a compact snapshot alongside AOF.
- **`redis-cache`** (host port 6379): `--maxmemory 256mb`, `--maxmemory-policy allkeys-lru`, `--save ""`, AOF off, and no volume.

The app reads `REDIS_QUEUE_URL` and `REDIS_CACHE_URL`. Containers reach the instances as `redis-queue:6379` and `redis-cache:6379`.

## Consequences
- ✅ Queued jobs survive a Redis crash (at most about one second of writes is lost) and are never evicted.
- ✅ Cache keys don't need a TTL for memory safety. Add TTLs only where data goes stale.
- ✅ A full cache can't block uploads.
- ⚠️ The broker isn't immune to running out of memory. A huge backlog fills `redis-queue`, and then enqueues fail **loudly** rather than losing data. Watch queue length (Phase 15).
- ⚠️ Postgres, not Redis, holds the truth about which jobs need doing. A message lost anyway (for example, the outbox gap in Phase 4) is recovered by the reaper reading Postgres, not by trusting Redis.
- ⚠️ The cache must be optional: if `redis-cache` is down, requests should run slower, not fail (design this in Phase 7/11).
- 🔁 Revisit if: queue volume needs a dedicated broker (RabbitMQ), or measurements in Phase 11 show `allkeys-lfu` gets a better hit rate than `allkeys-lru`.

## What I learned
- Docker controlled startup
- Removing docker volumes along with containers
- How to use volumes in Docker compose
- Perisitance and Eviction in Redis
