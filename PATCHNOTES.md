# EvenMoreFish 2.4.2-asyncdb — async database I/O fork

**Jar:** `even-more-fish-2.4.2-26.1-asyncdb.jar` (drop-in replacement, same config
format, same flyway schema, MySQL/MariaDB still supported).
**Base:** upstream `master` @ `a2ac1a5` (2026-07-09, includes everything in 2.4.1
plus the 2.4.2 development work — the data layer is identical in shape to 2.3.6).
**Branch:** `fix/async-database-io` on the `froquefy/EvenMoreFish` fork.

## Why

Production watchdog dumps (Purpur 26.1.2, remote MariaDB over VPN) showed the
server thread parked in `sun.nio.ch.Net.poll` under the MySQL driver inside
`Database.upsertFishStats` / `batchInsertFishLogs` / `upsertUserReport`, reached
from the fish-catch listener via `ImmediateSavingStrategy.save`. Every fish catch
did blocking database round-trips on the tick thread; any latency episode on the
VPN link froze the whole server for 10–15 s.

## What changed

All under `even-more-fish-plugin/src/main/java/com/oheers/fish/`:

| Class | Change |
| --- | --- |
| `database/data/DatabaseWriteQueue.java` | **New.** Single-threaded named daemon executor (`emf-db-writer-N`). All database writes are dispatched to it in FIFO order. Failed writes are logged, a failure does not kill the thread. On plugin disable it drains the backlog (up to 60 s) before the pool closes; writes submitted after shutdown run inline instead of being lost. |
| `database/data/strategy/ImmediateSavingStrategy.java` | Accepts an optional `Executor`; `save`/`saveAll` dispatch to it (the write queue in production wiring). The three users — fish stats, fish log, user reports — no longer block the server thread. |
| `database/data/strategy/BufferedSavingStrategy.java` | Same executor treatment for the write-through-on-create path **and** batch flushes (the `PlayerQuitEvent` flush used to run a blocking batch upsert on the server thread). Batches are snapshotted before hand-off. |
| `database/data/strategy/impl/*.java` | All five strategy impls take the shared queue. |
| `database/data/manager/UserManager.java` | Join-time user row lookup/creation moved onto the write queue (FIFO guarantees the row exists before any dependent write). A cache miss in `getUserId` still falls back to a blocking lookup — rare, and capped by the Hikari timeouts below. |
| `plugin/PluginDataManager.java` | Owns the queue; on shutdown flushes all data managers, drains the queue (60 s cap), then closes Hikari. Also now shuts down the fish log manager. Preloads: all global fish stats at init; the joining player's user report + all their fish stats rows on join (both on the queue, off-thread). |
| `database/Database.java` | New bulk loaders `loadAllFishStats()` / `loadAllUserFishStats(userId)` used by the preloads. Return `null` on query failure so a failed preload can never make an empty cache look authoritative. |
| `fishing/EMFFishListener.java` | On cache miss, consults the preload markers: if the scope was preloaded, a miss means "row does not exist" and no blocking existence SELECT runs on the tick thread. Fallback to the old on-demand load only if a preload failed. |
| `gui/guis/FishJournalGui.java` | Journal open used to run one blocking query per fish (`userHasFish` + two `get`s). Now answered from the preloaded caches when available. |
| `selling/SellHelper.java` | `createTransaction` / `createSale` INSERTs moved onto the write queue. |
| `database/connection/ConnectionFactory.java` | Hikari hardening: `connectionTimeout` 5 s (was default 30 s), `keepaliveTime` 1 min (keeps VPN/NAT connections warm), dropped the driver-level `validateBorrowedConnections` ping that doubled borrow-time validation (seen in spark profiles as `isConnectionDead → NativeSession.ping`). |

Also: `settings.gradle.kts` applies the foojay toolchain resolver (build-only).

### Semantics / residual notes

- Writes are now eventually consistent: a hard crash (kill -9, power loss) can
  lose the writes still queued at that moment. A clean stop/restart drains the
  queue (60 s cap — at normal latency that is thousands of writes; at 500 ms
  RTT it is roughly 40–60 writes, after which a warning logs the dropped count).
- Remaining blocking paths on the server thread, all rare and now capped at
  ~5 s by `connectionTimeout` + `validationTimeout`: `getUserId` cache miss
  (catch within milliseconds of join), user-report cache miss in placeholders /
  competition end for a player who somehow skipped join preload, and `/emf top`-
  style leaderboard queries that were already on-demand upstream.
- `messages`/config format, flyway migrations, table schema: unchanged.

## How it was tested

- **Unit:** 8 new JUnit tests (`DatabaseWriteQueueTest`, `AsyncSavingStrategyTest`)
  cover FIFO ordering off the caller thread, drain-on-shutdown, inline fallback
  after shutdown, failure isolation, executor dispatch and batch snapshotting.
  Full plugin test suite green.
- **Acceptance (local rig):** Paper 26.1.2 (build 63) + portable MariaDB 11.4.5
  behind a TCP proxy adding 250 ms each way (≈500 ms RTT), `database.type=mysql`
  through the proxy. A load-test companion plugin replicated the exact
  catch-path data-manager calls of `EMFFishListener.handleFishEvent` on the main
  thread, one catch per tick.
  - **Stock jar (master @ a2ac1a5):** 10 simulated catches blocked the main
    thread for **46.8 s total** — avg **4,684 ms**, max 7,115 ms per catch,
    10/10 over 55 ms; average tick time 4,411 ms (≈0.2 TPS). Join-path user
    init alone blocked 3.7 s. This reproduces the production watchdog freezes.
  - **Patched jar:** 50 catches with warm caches (the steady state a real
    player is in after the join preload) cost **3.0 ms total** on the main
    thread — avg **0.06 ms**, max 0.35 ms, **0/50 over 55 ms**; average tick
    time 20.5 ms, server at 20 TPS throughout. (A cold synthetic user's first
    catch pays one documented fallback load, ~1.5 s — real players never hit
    it because the join preload warms their caches off-thread.)
  - **Row integrity:** after the queue drained, all 100 fish log rows,
    10 user fish stats, 10 fish stats and `num_fish_caught=100` were present —
    identical totals to what the synchronous stock jar wrote.
  - **Shutdown flush:** a 10-catch burst followed by an immediate `stop` left
    ~30 writes queued; onDisable drained them in 46 s (under the 60 s cap) and
    post-mortem counts showed **110/110 rows — zero loss**.
  - **Competition:** an admin-started competition ran and ended cleanly; the
    `emf_competitions` report row was written through the queue and the
    leaderboard read path produced no errors.

## Rollback

Swap the previous jar back in. No schema or config migration in either
direction.
