---
layout: post
title: "CSI: Postgres — Did someone change my table??"
cover-img: /assets/img/sherlock.jpg
tags: [postgres, mvcc, track_commit_timestamp, pg_waldump, wal, logical-replication]
---


PostgreSQL has many small “hidden gem” features included (not to mention ~1K extensions adding a bunch more) waiting for
someone to notice them. Some are useful every day, some are niche, and some (e.g. `debug_*` settings)
exist only so that core developers could troubleshoot things without losing too much hair.

Today’s "heroes" are somewhat obscure but occasionally handy - [`pg_waldump`](https://www.postgresql.org/docs/18/pgwaldump.html)
and [`track_commit_timestamp`](https://www.postgresql.org/docs/18/runtime-config-replication.html#GUC-TRACK-COMMIT-TIMESTAMP).
Thought I'd give them a well-deserved boost - as recently stumbled upon a use case again. 

So for a very "skimpy" large legacy database some bytes were being saved (well, adds up indeed with a few thousand tables)
on auditing columns - which of course backfired on one good day and caused a lot of confusion. Namely, some calculations
started giving too many bogus results and was suspected that some relatively static parameter tables
had been fumbled with - which as mentioned, weirdly didn't have auditing columns from some decisions 10+ years back. But
ok, how to debug that? No full server audit logging was in place also, only DDL and slow
query logging, which didn't provide anything useful in this case as well. Restoring a backup (from which time even??) and
diffing 2TB quickly ad-hoc seemed too hard as well. Hmm...not many good options left on the database level or what? Enter CSI: Postgres! 🕵️‍♂️

# Unsung hero #1 - pg_waldump

Thanks to "insider knowledge" we luckily had a few hundred "suspect" table names to start with, that influence the calculations
the most + luckily WAL archiving was in place also (how else would you run Postgres??) - meaning one could try to convert
the table names to "filenode" names and "grep" through all the latest WAL entries and look for any UPDATE entries, looking
like something below:

```
rmgr: Heap        len (rec/tot):    163/   163, tx:    3986395, lsn: 2/E873C468, prev 2/E873C290, desc: UPDATE old_xmax: 3986395, old_off: 33, old_infobits: [], flags: 0x10, new_xmax: 0, new_off: 47, blkref #0: rel 1663/5/20467 blk 169834
```

With the grepping relevant part being `rel 1663/5/20467` (meaning to `tblspc/db/rel` according to the [documentation](https://www.postgresql.org/docs/18/pgwaldump.html)),
where `20467` is say one of our "suspect" table's *relfilenode*.

And low and behold - after throwing together a quick Bash loop plus an hour or so of runtime (looping 300x over a few dozen gigabytes
of WAL data) there was a match - indeed there were changes that shouldn't have taken place in a few of the tables!

Now it was of course easy to do a PITR restore on the side (the WAL LSN numbers could be converted into a timestamp as well
as some monitoring was in place storing timestamped `pg_current_wal_lsn()` converted into a numeric via `pg_wal_lsn_diff()`) and
extract the old correct table contents.

*PS!* A side learning from here - good monitoring can turn out to be extremely valuable for random ad-hoc tasks! Here
though as an alternative the approximate change time can be seen from archived WAL timestamps as well. 


# The more humane way for the future - track_commit_timestamp

To make life easier in the future for similar cases (as source of error was still not known - time to start shaking through
senior developers and DBA's with direct production access as well;)) 2 things were done:

1. Start development and rollout of common auditing columns for all of those parameters type tables - but this takes a bit of time of course in a larger org ...   
2. Immediately enabled the PostgreSQL `track_commit_timestamp` parameter - so that when a mysterious table change is suspected on a
  table with no auditing columns, someone could ask: “So… when was data last modified in this table? Or even - for which exact rows recently!”

## What does `track_commit_timestamp` actually do?

The setting has been around since **Postgres 9.5**, but is **disabled by default**, thus doesn't have many "friends". And — if you
actually design your schemas correctly (have proper `created_at` / `updated_at` columns) and are not maintaining any
Logical Replication replicas also, you shouldn't miss it as well to be honest.

In short it's simple - when enabled, PostgreSQL starts remembering the **timestamp when each transaction was committed**,
which you can later query conveniently via SQL 🥳

**BUT** this information is stored only semi-persistently — typically available for **days or weeks**, depending on
vacuum and freeze settings and amount of transaction churn happening. Thus, it is not meant to be a permanent audit log - but
nevertheless can still be useful for some forensics as we saw.

The setting also **requires a server restart**!


## How do I find when a table was last modified using commit timestamps?

Let’s assume we have a table `pgbench_accounts` that was designed back in year 2000 when someone thought extra timestamps were
not fashionable and just drain precious resources:

```sql
krl@postgres=# \d pgbench_accounts
              Table "public.pgbench_accounts"
  Column  │     Type      │ Collation │ Nullable │ Default 
──────────┼───────────────┼───────────┼──────────┼─────────
 aid      │ integer       │           │ not null │ 
 bid      │ integer       │           │          │ 
 abalance │ integer       │           │          │ 
 filler   │ character(84) │           │          │ 
Indexes:
    "pgbench_accounts_pkey" PRIMARY KEY, btree (aid)
```

But now with *track_commit_timestamp* enabled we can query for example the timestamp of the most recent (new or updated)
tuple by looking at its `xmin` (the inserting transaction ID):

```sql
SELECT max(pg_xact_commit_timestamp(xmin)) FROM pgbench_accounts;
```

Boom, you now have the **last commit time affecting that table**. Worth remembering as well - this gives you **transaction commit time**,
not row-level update time. Close enough probably for most practical OLTP cases though. Also mind that the current query
will result in a full table scan, but this could be avoided if we have some indexes + knowledge on what we're looking for.

For an instance level indication (remember - Postgres WAL and transaction counters are shared between databases) take
a look at `pg_last_committed_xact()` instead.


## Caveats of *track_commit_timestamp*

### The MVCC caveat

Some fine print - commit timestamps are **not kept forever**.

Why? Because of PostgreSQL’s MVCC — the *éminence grise* - responsible for many annoyances and good features as well.
In short: old entries are recycled when the freezing (VACUUM FREEZE) horizon advances those "xid"-s. PS The horizon freezing
aggressiveness can be controlled mainly by `autovacuum_freeze_max_age` and `vacuum_freeze_table_age` but be aware of [risks](https://www.postgresql.org/docs/18/routine-vacuuming.html#VACUUM-FOR-WRAPAROUND)
of delaying it too much ☢️

In any case due to the freezing aspect commit timestamps are not a great fit for:

* Audit trails
* Logging
* “Who touched this row in 2018?”

### Extra disk usage

Once enabled, Postgres starts writing some bytes into `$PGDATA/pg_commit_ts`, namely 8+4 bytes for each transaction
according to [this](https://www.postgresql.org/message-id/flat/1b2e2697-76fc-7d8f-1ca6-588093531d7d%40anayrat.info#9a705c534f119fad3b6b09e13cd310b7)
hackers thread.

Thus you need to also plan ahead and ensure some extra disk space! Normally (with default settings) to the tune of around
a few GB so shouldn't be a real problem, but maximally already noteworthy 20GB when playing around with `vacuum_freeze_table_age`.
Full docs [here](https://www.postgresql.org/docs/18/routine-vacuuming.html).

### Performance impact?

Short answer: no need to worry about performance on modern hardware on workloads that do some actual work and just don't COMMIT for fun.

But Postgres of course does an extra bit of work on every commit once the setting is enabled:

* Capture timestamp (timing overhead can be tested with `pg_test_timing`)
* Write into `pg_commit_ts` SLRU (which will be persisted on CHECKPOINT)
* WAL-log the timestamp record

Google didn't sadly help much with some actual performance numbers / tests, except "I didn't notice any performance impact"
and "I don't think the performance penalty will be terrible"....so had to roll my own as well out of curiosity and to
be on the safe side. For that threw together a quick test [script](https://gist.github.com/kmoppel/d16d188f992aaebc29c800e59432bbbf#file-test_track_commit_timestamp_perf_impact_v2-sh)
and run it on my old home server + my more modern workstation, and indeed the results were OK I think:

| CPU                                                     | Disk                                            | pgbench --skip-some-updates TPS penalty |
|---------------------------------------------------------|-------------------------------------------------|:------------------------------------------|
| Intel(R) Xeon(R) CPU E3-1225 v6 | Intel SATA SSD DC S3500 480GB (from year 2013!) | ~3%                                       |
| AMD Ryzen 9 5950X                                       | ADATA SX8200PNP 512GB NVMe SSD                  | ~0.6%                                     |

Note that *pgbench --skip-some-updates* mode is relatively commit heavy, so I guess not much reason to worry indeed unless you are running:

* Extreme commit rates with very simple transactions / queries 
* Systems targeting ultra-low latency 
* Ancient spinning disks

### Operational gotcha - pg_upgrade forgets your timestamps

One more thing worth knowing about - as of PostgreSQL 18:
 
> `pg_commit_ts` folder is **NOT migrated by pg_upgrade**

Meaning - after a major upgrade your historical commit timestamps are gone! BUT...seems there is a patch proposed to
fix this for Postgres 19: [https://commitfest.postgresql.org/patch/6119/](https://commitfest.postgresql.org/patch/6119/)

So fingers crossed 🤞


## A Logical Replication "freebie" 

If you’re using Logical Replication, the commit timestamp setting automatically becomes more useful than just in case waiting
for some mishap - as some replication (data) conflicts can only be detected or logged on the subscribers when it's enabled!

From the [docs](https://www.postgresql.org/docs/18/logical-replication-conflicts.html):

> PostgreSQL can log and detect certain replication conflicts only if `track_commit_timestamp` is enabled.


*PS - feel free to [contact](https://kmoppel.github.io/aboutme/) me if need a bit of help with Postgres - I've put in my
20K+ hours with many Postgres-heavy top tier businesses in Europe so that you don't have to re-invent the wheel*
