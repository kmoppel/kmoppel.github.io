---
layout: post
title: No, you don't necessarily need extensions to compact Postgres tables
cover-img: /assets/img/plain_sql_compact.jpg
tags: [postgres, vacuum, bloat, pgbench]
---

Broadcasting a quick tip from a real-life "tasklet" where a customer was genuinely surprised why a seemingly logical fix
didn’t show up on Google / LLM-s. So let’s see if we can improve on that situation.

So in short a large central table was getting too bloated (fragmented) and query timeouts started to be a problem.
Lads tried VACUUM FULL during "low hours"…but it did not pan out still, too much locking.

And to make it worse - for some weird reason their DBaaS provider of choice didn’t provide [pg_repack](https://github.com/reorg/pg_repack)
as well. Which actually is a solid battle-tested tool, available on all top clouds. Not to dream of even better
[pg_squeeze](https://github.com/cybertec-postgresql/pg_squeeze) extension …

So what to do? Take the downtime hit still and lose in money / customer trust? Well, luckily there are some options still …

# The pure SQL “online compaction” way

A very little known approach to (try to) reduce bloat is to:

* “Live migrate“ top of the heap rows to the bottom of the heap, using plain UPDATEs
* Call VACUUM often to try to release the now empty top of the heap disk chunks back to the OS
* Re-index all indexes, as they most-likely have many holes in them after such a re-write

Be warned though - this approach is pretty resource-intensive! And can introduce some user-visible locking effects, as
the heap truncation step requires an `ACCESS EXCLUSIVE` lock on the table, and a lot of them in this case.

Also - when having triggers on the table, this approach might bring side-effects if care is not taken - especially on
some managed services, where one can’t transparently disable the triggers via the `session_replication_role`.

In any case your best bet probably on that road is to use the [pgcompacttable](https://github.com/dataegret/pgcompacttable) utility.

# Logical Replication

If your "troublemaker table” already makes up for the majority of your DB size - a full blown LR clone +
a switchover might make the most sense. And it’s not that scary even these days as there’s quite some tooling and documentation
available.

If a full LR switch seems too much though, one can also consider “in database” LR sync for a single table 🤯! But be
warned - you’re pretty much on your own with that approach, as sadly it’s not supported natively by Postgres.

But in short it’s doable! With some custom pgoutput, wal2json or test_decoding output parsing and back-feeding - plus
the messy initial data “snapshotting” with replication slot pre-creation, snapshot exporting and dump-reloading with pg_dump.
Or trying something more full-blown like Debezium, which on paper seems to support Postgres-to-Postgres sync as well.

# The trigger + snapshot + merge way

So what I went with in the end was a simple and foolproof approach basically - which just needs a tiny bit of application
downtime (if implemented correctly 🤞)...and assumes we run it during some relatively low activity time period, where only a
fraction of the data changes.

So how does it work? The steps are approximately like that:

## The general process

1. Ensure some “last modified” column exists on the “to-be-compacted” table, and is indexed for fast “give me all last changes” queries.
  - Create one for the duration of the process if missing, and use a partial index  
1. Install a trigger to log deletions (if any). Not needed in case of timestamped soft-deletes.
1. Create a new clone table with identical structure to replace the old bloated table.
1. Fill / snapshot the new table with current data from the “old” table via a plain SELECT - which leaves table bloat behind.
  - Yes, we start with a stale snapshot here! Don’t worry about the non-perfectness, it will be taken care of with a “merge”...
  - For humongous tables you might want to chunk it still somehow a bit
1. Create all the index-backed constraints and normal indexes as fast as possible.
  - Increasing max_parallel_maintenance_workers and maintenance_work_mem
  - On the clouds might want to actually temporarily upscale for that step (given doesn't incur much downtime)
1. Stop application writes / exclusively lock the table.
1. “Fix” our snapshot - as it’s already stale from normal business mutations. By:
  - Deleting rows not anymore in “old” - the ID’s were collected via the installed trigger
  - Add newly added rows and update changed columns - tracked via the “last modified” column from step 1.
1. Add back FK-s and triggers, if any.
1. Switch the tables and re-open to app connections.

## Assumptions / prerequisites / tips

* The approach only works well if a fraction of the data changes during our snapshotting + constraints / indexes rebuilding time window
* The approach has worked fine for me - and only gets problematic when there are Foreign Keys pointing to large tables
  with mutating ID-s, i.e. some FK value in our initial snapshot is actually already gone. Then FK-s can’t be created
  after the initial data snapshotting phase and some extra NOT VALID juggling is required to reduce locking.
* It is always a good idea to test out such things beforehand on some similarly specced test system / PITR-clone.
  Usually we know how many rows per minute we change on average, so that we can easily simulate expected mutations and
  gauge the approximate “merge”, i.e. downtime, duration.

## Possible speedups compared to VACUUM FULL

For you to better visualize this approach, and to provide some approximate speedup number - I also created a simple
executable Bash [script](https://gist.github.com/kmoppel/8dedcf01917e3fbc33cc31d48dbd3e0f) based on pgbench, with some
extra indexes added for a more realistic use case.

And the **speedup** number for a smallish 19GB table (3 indexes), with ~0.1% of row mutations after the initial snapshot -
was **~200x** on my workstation! And I have good disks - for a typical managed service the win will be even bigger.

```
bash pg_repack_in_pure_sql.sh
…
* MERGING - DOWNTIME START *

INSERT 0 60240
DELETE 7933

Merged in 1 s

ROWS_NEW 120000048, ROWS_OLD 120000048
SUM_NEW 5742231, SUM_OLD 5742231

* VACUUM FULL *
VACUUM

VACUUM FULL duration 227 s

```

### Postgres MERGE ?

Some more knowledgeable Postgres users looking at the sample script might wonder why I didn’t use the PostgreSQL
[MERGE](https://www.postgresql.org/docs/current/sql-merge.html) command?
Isn’t it exactly built for such data rectification tasks? With v17 additions, covering also deletes, indeed - would be
very convenient...but sadly way slower in practice - and the documentation mentions that as well.


*PS - feel free to [contact](https://kmoppel.github.io/aboutme/) me if need a bit of help with Postgres - I've put in my
20K+ hours with many Postgres-heavy top tier businesses in Europe so that you don't have to re-invent the wheel*
