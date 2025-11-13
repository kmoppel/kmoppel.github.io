---
layout: post
title: Postgres, Kafka and event queues
cover-img: /assets/img/pg_kafka.jpg
tags: [postgres, kafka, queues, bloat, ramblings]
---


After stumbling on a pair of interesting blog posts — [You Don’t Need Kafka, Just Use Postgres (Considered Harmful)](https://www.morling.dev/blog/you-dont-need-kafka-just-use-postgres-considered-harmful/) — somewhat in the style of good old “flame wars” (which are increasingly rare these days)
in the recent [Postgres Weekly](https://postgresweekly.com/issues/623), as a response to a previous article — [Kafka is fast -- I'll use Postgres](https://topicpartition.io/blog/postgres-pubsub-queue-benchmarks)
— on using Postgres for “Kafkaesque” business, I felt the urge to chip in a bit.

But first off — I’d like to laud the authors of both pieces. They’re well-argued reads with a crazy amount of good tidbits
and food for thought. I especially liked that the original one tried to be open and repeatable and actually tested things.
Gunnar's take was maybe a bit too morbid for my taste, of course 🐘

To recap — the main question in the debate was whether Postgres is generally *“good enough”* to implement a low-to-medium
volume event queue or even a pub-sub system. The general sentiment from [Hacker News](https://news.ycombinator.com/item?id=45747018) readers
at least was that unless scale truly demands Kafka, Postgres is indeed good enough — and many teams plain overestimate their
scaling needs and don’t actually need Kafka’s distributed complexity.

**Spoiler:** there’s obviously no definitive answer as to when one should use a “proper” database for something — and there
sure are reasons why we have so many purpose-specific databases: relational, event logs, analytical column stores, key-value, time-series, ledger, graph, hierarchical, document, blob, text search, vector, …

Anyway, below are some thoughts that came to mind — I can’t go too deep on Kafka though, as I’m just not qualified enough.


# On Running Postgres Tests

* **Test duration:** 2-minute test durations are never a good idea under any conditions — especially in the context of
a “permanent” (until Postgres copes) queuing solution, which all boils down to autovacuum and bloat effects. At minimum,
the test should run until autovacuum has kicked in a few times, so that TPS / RPS effects could be observed on some graph. 

* **Synchronous replication:** Running the test in synchronous replication mode is actually a fair idea — to be closer
to some other cluster-native software like Kafka. But with two replica nodes one should still run in `ANY` mode instead
of `FIRST` if performance is a goal.

* **Config tuning:** Super important for high I/O churn instances!
  More aggressive autovacuum settings are a must for queues (which the test set did!), but other things matter too — for example:

  * Better TOAST compression: `default_toast_compression=lz4`
  * Better WAL compression: `wal_compression=zstd`
  * Backend I/O shaping: `backend_flush_after=2MB`
  * Possibly enabling group commit
  * ...

* **Fsync considerations:** Even mentioning `fsync` is somewhat criminal 🙂
  You get similar benefits from `synchronous_commit=off` for selected use cases — where Postgres is not the original
event source - is definitely valid, given some error detection / retry is in place for the rare Postgres or node crashes.

* **Performance observations:**
  The actual queue test result numbers looked quite low to me. For example a message rate of 20K msg/s at 20 MB/s write throughput on
  a 96 vCPU / 192 GB RAM machine could definitely be improved several times over. Postgres can handle far higher workloads than people assume!
  This reminds me of a good recent article on some out-of-the-box thinking with Postgres by my ex-colleague Ants:
  👉 [Reconsidering the Interface](https://www.cybertec-postgresql.com/en/reconsidering-the-interface/)

* **Transactional benefits:**
  One important reason to prefer an *all-in-Postgres* approach (again, if scale for few next years allows) — if the
  initial system of record was Postgres for the originating fact — is that you’re guaranteed automatic all-or-nothing behavior with transactions,
  if using say `postgres_fdw`. This is much harder to achieve in a flow like: `app → Postgres → DB X / Kafka ← consumer app → another Postgres`

* **Postgres version matters:**
  When running actual tests, one should also mention the exact Postgres version tested — Postgres has improved a lot in the
  most recent version, especially for I/O-intensive applications like queuing. (From the code, it seems v17 was used.)

* **WAL sync method:**
  Setting `wal_sync_method=open_datasync` is a slippery slope. Although `pg_test_fsync` can report that `open_datasync`
  is faster than `fdatasync` (the Linux default), in real life — where occasional larger writes occur — `fdatasync` often wins.
  In essence, `open_datasync`’s `O_DSYNC` flag means “synchronous per write,” enforcing data durability on every write
  call, whereas `fdatasync` allows PostgreSQL to control more precisely when the expensive sync occurs.

  This effect can be observed when running `pg_test_fsync`.
  With a simple double-block (16 kB) write + sync, `fdatasync` already pulls ahead.
  PS The test below was done on an equivalent `c6i.xlarge` machine + 9000 IOPS EBS volume as in the original test.
  The commit/fsync performance numbers for such hardware, by the way, are criminally low — I wouldn’t recommend
  running real queue apps on that.

  ```
  Compare file sync methods using one 8kB write:
  (in "wal_sync_method" preference order, except fdatasync is Linux's default)
          open_datasync                      1093.933 ops/sec     914 usecs/op
          fdatasync                          1094.303 ops/sec     914 usecs/op
          fsync                               601.509 ops/sec    1662 usecs/op
          fsync_writethrough                              n/a
          open_sync                           601.358 ops/sec    1663 usecs/op

  Compare file sync methods using two 8kB writes:
  (in "wal_sync_method" preference order, except fdatasync is Linux's default)
          open_datasync                       551.779 ops/sec    1812 usecs/op
          fdatasync                          1019.174 ops/sec     981 usecs/op
          fsync                               654.387 ops/sec    1528 usecs/op
          fsync_writethrough                              n/a
          open_sync                           349.305 ops/sec    2863 usecs/op
  ```

* **Bloat awareness:**
  With queuing/pub-sub in Postgres, one **must know** the ins and outs of Postgres bloat and containment techniques — as
  one could say it’s one of the biggest architectural downsides of Postgres.
  For low-TPS apps, making autovacuum very aggressive can be enough.
  For high-volume or high-velocity cases, some nightly reindexing, repacking, or squeezing might still be necessary.
  Better yet — avoid unnecessary work altogether: don’t `UPDATE`, use partition dropping instead.

* **Design caution:**
  Carrying over concepts from other systems 1-to-1, like Kafka’s ordered offsets and their maintenance, is a bit of a red flag — they’re rarely useful artifacts at the user-facing API level.

# Postgres-Native Queuing Solutions Tips

Using something boxed like [pgmq](https://github.com/pgmq/pgmq) is probably better than rolling your own if you’re not too deep into Postgres.
It’s not perfect though — and from a pure performance standpoint, you could still squeeze out some extra TPS with a custom solution.

**My complaints with pgmq specifically:**

* Payload storage is only in `JSONB` — no plain JSON option, which is often a few times faster for larger payloads or high row counts.
  The tables can also be smaller (depending on the data).
  Generally speaking, JSONB has little point if it’s not indexed or used for search — that’s where its effectiveness comes from.

* No `FILLFACTOR` setting options.
  Although pgmq isn’t HOT-update friendly anyway due to `vt` (visibility timeout) being indexed, one could still gain a bit there.

* There’s no “peek” mode yet (though there’s a PR).
  And `pgmq.read()` always does an `UPDATE`, which generates bloat — the *enemy* of Postgres queuing systems.
  Ideally, visibility locking should happen in another table to minimize bloat on the main queue table.

In short, for high-performance setups I’d recommend looking at good old [pgq](https://github.com/pgq/pgq), which has many
niceties like lock-free queue reading (no `SELECT FOR UPDATE SKIP LOCKED`, which actually writes to heap rows!) via background batching “tick” processes.
Also, pgq does native table switching to keep bloat at bay, whereas pgmq only supports that via **pg_partman** — yet another
extension (read: long-term maintenance risk).

A good trick if you go the Postgres route — but might need to scale up later — is to **hide the queue details behind a database-level API!**
Meaning at minimum a few PL/pgSQL functions like pgmq provides.
Such an API allows you to transparently migrate later to a more sophisticated partitioning strategy once a naive single-table approach hits a wall.

Another idea - once can also actually employ Postgres Logical Replication / CDC to read the incoming events - so that one
doesn't have to poll blindly. And to help with performance here (LR can get costly) - in recent versions one can specify
pretty exactly on which operations (INSERT) or columns (WHERE) to listen on. 


# But Still — Who “Wins”?

As humans, we’re always looking for simplifications.
Having seen and implemented some pretty crazy things on top of Postgres, I tend to say that Postgres is *mostly* good
enough for many purposes, for many years.

For queuing — with some work, no doubt.
For pub-sub — less so.

And honestly if the below mind-boggling statistics from Stanislav's blog post, in context of comparing to Kafka, are
even remotely correct - the bar is very low to begin with.

> A report by RedPanda found that ~55% of respondents use Kafka for < 1 MB/s.
> Kafka-vendor Aiven similarly shared that 50% of their Kafka deployments
> have an ingest rate of below 10 MB/s.

Of course, if you already know you’ll need to scale like crazy, go with something else.
Kafka, while not directly comparable, is good but adds operational overhead for sure — at least based on experience from my previous startup.
The issues we had with the Strimzi Kubernetes operator alone dwarfed any Postgres management and scaling woes.

You do get quite a bit in return though — from end user point of view most importantly more limited failure impact and
much faster recovery times than with a typical HA Postgres cluster.

Another aspect in favor of PostgreSQL though — **there are probably two orders of magnitude more PostgreSQL installations
out there than Kafka (or X)** — meaning it’s *very* battle-tested.
That alone lets me sleep more peacefully at night.
The sad part, of course, is that it makes my resume look pretty boring :)

# Parting Thoughts

If PostgreSQL is already the main database in your organization, why not test it's limits - properly, with a high-volume long-term test setup?

As a heavy KISS practitioner / proponent / chronic under-engineer, I’d weigh the pros and cons very carefully before introducing new technologies to the team.
Chances are that getting a production-quality Postgres instance up and running takes only a few clicks or some IaC copy-paste — and then you can focus on data flows, good schema, and query design, i.e., the stuff that really matters with relational databases!

That’s a wrap — another half working day lost to the internet instead of billable customer work on an Oracle to PostgreSQL migration project 😄
Is there any cure for this? Aarghh. Hope someone at least feels slightly more informed after reading it.

