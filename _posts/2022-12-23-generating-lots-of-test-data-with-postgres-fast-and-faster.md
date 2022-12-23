---
layout: post
title: Generating lots of test data with Postgres, fast and faster
tags: [postgres, testing, testdata, pgbench, generate_series]
---

After having worked many years with databases, some things become kind of second nature, so that you don’t think about it
really - if there’s a need, you just type in some commands on the console and move on with other things. One of such things is for example
generating lots of dummy test data in an efficient way. For DB engineers a very common task actually! I think a week doesn't
go by where I don’t run _”pgbench \-\-init \-\-scale X”_ at least once.

But as I recently, to my surprise, found out at work that such simple "no thinking required" operation might actually not
be so obvious for generally database-aware people still. So a small primer here on a few ways to generate gigabytes of test
data - hope you find it useful.


## Why we need test data generation at all?

First maybe let's tackle the “why” part? Why is it sometimes handy to be able to grow your Postgres database quickly? Well some good reasons for doing that might be:

* Validate your indexed / batch lookup latencies for that time in the future when the data doesn't fit into RAM anymore.
* Check if some DDL operation wants to do a full scan unexpectedly. As of now one can’t sadly EXPLAIN DDL, i.e. altering of data types and such.
* Testing if your "low on disk" alerting works. For managed DBs it’s the only correct way I guess. For “on-prem” would be
  also just fine to “dd” zeroes directly to the volume, hard to beat that in simplicity and speed.
* Measure block latency / bandwidth change when the dataset is growing into Terabytes and Terabytes. As on cloud platforms
  disk fragmentation is a very likely thing to happen, as they do all kinds of optimization tricks behind the scenes, so
  better to verify that if starting with some Data Warehouse project. (Azure Postgres Single Server springs to mind here immediately  - that now thank god is being deprecated)
* Checking if volume auto-resizing works fast enough. As I've seen some cases where the cloud provider-side resizer is actually
  a bit too laggy and you get to see the nice "No space left on device" Postgres error, if writing too heavily.
* How much space / $$ would backups and snapshots allocate for huge DBs after possible compression / deduplication steps
  from the provider side. Note that here for accuracy ideally the data should already look a bit like your domain.
* How much time would it take to do a `pg_dump` or PITR restore clone from a future life-sized DB. Note though that for PITR
  test cases you can't take the UNLOGGED performance optimization shortcut with pgbench, but need to push everything through WAL, making the process a bit slower.


## Level 0 - "low hanging" SQL

Everybody in the “data sphere” knows SQL, right? And if to add the PostgreSQL specific `generate_series()` sequence / row generation
function into the formula, we have enough to generate a bit of dummy data! And indeed - for the simplest cases, like checking
if some DDL operation wants to sneakily do a full scan, I use that:

```sql
CREATE TEMP TABLE t AS SELECT generate_series(1, 3e7) x;
```

I’ve memorized that 30M rows generates a 1GB table in a few seconds - and that’s enough data for example to recognize the
fact that some DDL command is not instant, i.e. it does a full scan, which you generally want to avoid with a lock on the table.


## Level 1 - pgbench vanilla “init”

When things get more serious though, we want a more proper tool I guess - something that is easily customizable, shows
progress, and where target data sizes can be predicted / calculated and where we can quickly switch to benchmarking mode
with latencies nicely analyzed. For Postgres one typically then stumbles on something like:

`pgbench --init --quiet --scale 5000 $DB`  or just `pgbench -iq -s 5000 $DB`

This one is a no-brainer I guess and not much commenting needed - everyone who works more with Postgres most probably has that in their toolbox.

A common question mark here though for beginners is the “\-\-scale” number - what the heck does that number 5000 actually mean?
The documentation says: 1 “scaling factor” = 100k banking schema account rows (plus some auxiliary rows, which can be dismissed though in the data size sense).

But what does it mean in terms of the output dataset size? Well, around 73GB :) How do I know that? Yeah, a bit of a bummer
actually as it requires some knowledge, or we could just apply a [known formula](https://www.cybertec-postgresql.com/en/a-formula-to-calculate-pgbench-scaling-factor-for-target-db-size/)
instead. Sadly the target size can’t be specified directly (as it depends on a few things).

So to have a starting baseline for our test data generation exercise for ~73 GB of data, we run:

```
pgbench -iq -s 5000 postgres
...
done in 873.80 s (drop tables 0.00 s, create tables 0.00 s, client-side generate 494.51 s, vacuum 0.92 s, primary keys 378.37 s).
```


## Level 2 - less “logging” please

The big downside with the vanilla pgbench init, is that we’re generating a lot of transient WAL files (also known as transaction logs),
thus doing “double-writing”! Surely not optimal for us, as testing commonly is not so critical that we want trade crash safety for
speed - “UNLOGGED” tables to the rescue! A very underused feature of Postgres by the way, for all kinds of data staging and
temporary multi-step calculations as it’s generally better than temp tables - gets autovacuumed / analyzed, other users / worker
process can see it, doesn’t bloat the Postgres catalog if executed frequently enough.

```
pgbench -iq -s 5000 --unlogged postgres
...
done in 524.78 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 177.13 s, vacuum 1.19 s, primary keys 346.44 s).
```
Much better - **a 40% boost**!


## Level 3 - data only

Not bad already, but obviously we’re just getting started :smile:
Why spend time waiting for the CPU crunching on index creation, sorting data? When, say, we’re actually interested in just
filling the disk in this case.

For such cases since quite some time (v11+) we can actually explicitly set the data generation [steps](https://www.postgresql.org/docs/current/pgbench.html#PGBENCH-INIT-OPTIONS)
we want to take in the pgbench init phase with the `--init-steps/-I` flag! We just need the first 3 steps - “dtg”.

Note though that when you have a strict size target, you need to up the “scaling factor” a bit to compensate for the index size
when leaving out the primary key indexes. Some +15% should roughly do it. I’ve been pretty dismissive about it mostly though,
but will account for that in the results summary table below for correctness here.

Also note that we’re leaving out the vacuuming part - but it will most probably happen very soon in the background still,
but the resulting disk size changes from that are negligible for a freshly initialised schema.

```
pgbench -iq -s 5000 --unlogged -I dtg  postgres
...
done in 181.22 s (drop tables 0.87 s, create tables 0.06 s, client-side generate 180.28 s).
```

More wow - **roughly a 3x boost while just having 15% less data!**

PS! If you indeed start missing the omitted index later, we can add it back anytime with `pgbnech -i -I p`. Or sometimes even for
huge DBs, I make it explicitly a BRIN index instead - to test kind of the worst case "key" fetching, as pgbench generated
data has perfectly ascending ID-s, if not yet modified. 


## Level 4 - hitting IO even harder

We’re not done yet :smirk: There’s still one pgbench flag left to set, in case there’s a doubt that we haven’t saturated our IO
(or network bandwidth for remote DBs) yet - we can further reduce the CPU bottleneck a bit! Specifically by instructing
Postgres to pack the row data into data blocks / pages with maximum amount of “air bubbles” aka [“fillfactor”](https://www.postgresql.org/docs/15/sql-createtable.html#id-1.9.3.85.6.3.2)!
Minimum allowed value there is 10 - meaning 90% of data block space will be reserved for future INSERT-s or UPDATE-s and the CPU will
have much less work to do converting our input data to binary tuples, so that IO will have the centre stage.

Note here that to keep the target data size similar to our starting point we need to now lower the scaling factor also by
a factor of 10! We’ll lose just a few percent of precision in the process but it’s dismissable again.

```
pgbench -iq -s 500 --unlogged -I dtg --fillfactor 10  postgres
…
done in 157.47 s (drop tables 1.09 s, create tables 0.03 s, client-side generate 156.34 s).
```

Benissimo! **Another ~15% reduction** in data generation runtime. But at the same time we can see that we’re reaching the
diminishing returns zone already…all parties must come to an end still seems.


## Level 5 - parallel streams

So we’re at the end…one last try to speed things up - again, assuming we haven’t yet squeezed the absolute 100% out of our
storage system, which generally tends to be the bottleneck with databases.

If you happened to look at the Postgres process tree or “top” output during our test data generation in the previous step,
you might have noticed that we had a single process churning at 100% CPU! Could we improve things by having another processes?
More than 2 btw generally should not be needed, as we commonly get IO-bound already then for sure.

So let’s throw in a new test DB + a pgbench init runner in the background!

```
createdb bench
pgbench -iq -s 250 --unlogged -I dtg --fillfactor 10  bench &
pgbench -iq -s 250 --unlogged -I dtg --fillfactor 10  postgres
wait
...
done in 151.03 s (drop tables 0.00 s, create tables 0.11 s, client-side generate 150.92 s).
done in 150.37 s (drop tables 1.46 s, create tables 0.14 s, client-side generate 148.77 s).
dropdb bench
```

So did we win? Meh...not sure, **just a tiny ~4% improvement**. But…at least we can now be definitely sure that our IO is maxed out!

PS! Note here that if your hands are tied in creating a new throwaway DB, **you could also just create a new schema and
set the “search_path” accordingly**, shouldn’t matter from IO performance point of view:

```
psql -c "create schema if not exists pgbench1" postgres
PGOPTIONS='-c search_path=pgbench1'  pgbench -iq -s 250 --unlogged -I dtg --fillfactor 10  postgres &
pgbench -iq -s 250 --unlogged -I dtg --fillfactor 10  postgres
wait
psql -c "drop schema if exists pgbench1" postgres
```


## Results overview

To make things more relatable / useful for most folk at our company, I decided to throw in the Cloud aspect also - as generally,
the slower the disks the more you could win with such tweaking.

All times in seconds.

|                                                                   | Default init  | +Unlogged | \+ Data only | +FF 10 | \+ 2 streams | Total adjusted improvement (\*) |
|-------------------------------------------------------------------| ------------------------- | --------- | ------------ | ------ | ------------ | ------------------------------- |
| My local workstation (512GB Sata SSD)                             | 873                       | 524       | 181          | 157    | 151          | ~5x                             |
| A typical managed Cloud instance (4vCPU, 16GB RAM, 512 GB volume) | ~2700                     | ~1500     | 430          | 340    | 337          | ~7x                             |

\* remember, we generate 15% less data compared to the 1st run when leaving out the primary key index.

Local Postgres version was v15, Cloud ones v14 with the pgbench node in the same AZ. Average of two instance runs, each on
separate top tier cloud hosting providers, let’s just call them the “blue” and “green” clouds 😏


## Bonus level stuff for DB “athletes”

During my on-prem consulting days I’ve also had the luck to work with some real beasts of databases - meaning tons of parallel
IO via special hardware - so that a few CPU cores generating data are actually unable to saturate that. What a nice problem to have!

In that case you need more flexibility in choosing the correct level of parallelism and fiddling with multiple pgbench schemas
is not so cool anymore - especially if you want to later run some actual queries over the humongous test table, to get some
latency approximates under heavy load. So this approach (with varying details) is what I’ve applied in such cases:

```bash
SCALE=5000
FF=10
CHUNK_SIZE=10000
CLIENTS=4 # Lvl of parallelism
TX_PER_CLIENT="$(((SCALE*100000/$FF)/CLIENTS/CHUNK_SIZE))"
pgbench -iq -I dtg --unlogged -s 10 --fillfactor 10 --partitions $CLIENTS --partition-method hash postgres
SQL=$(cat <<EOF
\set part_id :client_id + 1
INSERT INTO pgbench_accounts_:part_id SELECT * FROM pgbench_accounts_:part_id LIMIT $CHUNK_SIZE;
EOF
)
echo "$SQL" | pgbench -n -f- -c $CLIENTS -t $TX_PER_CLIENT postgres
```

So basically - creating a bunch of partitions somehow, a bit of seed data, and then repeatedly re-inserting seed data directly back
into **sub-partitions, bypassing the parent table** so that each worker process has it’s “own” table - meaning less locking
on extending the table and stream-like sequential writing on all datafiles, setting the stage for maximum IO performance!
Here you might also wonder why we're re-inserting the same data basically? Well, just because I've found it's a faster than
generating fresh, especially random datam, from a generator function. 

Also this approach is very handy if you **want to fill the database exactly to the brim**, in a persistent way! Meaning if you
just do a huge `pgbench --init --scale $gazillion`, and overdo a bit and run out of disk space, the whole dataset gets
discarded on erroring out! But if we work with 10k or 100k chunks we just fail on that last chunk and are good.

PS! Note though that sometimes real unique ID columns are required, so you’d want to alter the original schema a bit, set
up an identity column or sequence or something - adding some artificial ID in “post” ruins the whole point of “fast”.

PS2! Also note that this approach doesn’t make any sense for managed cloud offerings and also weaker on-prem machines, as you’d get throttled / saturated quickly
on something and it would be slower than plain unlogged + fillfactor in the end, as adding data with streamed COPY is a bit more efficient than via batched INSERT.


## Summary and additional remarks

So what have we learned? I think we could formulate a rule of thumb such that:

*If you’re not on some very expensive or IO-beefy hardware, the simplest and quickest way to fill a Postgres database with
some dummy data that you actually don't want to touch later on, or need custom indexes anyways, is to run:*

`pgbench -iq -s $target_scale --unlogged -I dtg --fillfactor 10  postgres`

Where the *$target_scale* can be calculated using this little [snippet](https://jsfiddle.net/kmoppel/6zrfwbas/).


A few more thoughts:

* There are heaps of other ways and bennchmarking tools to generate or load test data, that might have more flexibility (Sysbench, HammerDB
  for example) - but as **pgbench comes boxed with Postgres and is dead simple and fast**, so I tend to prefer that.

* When you already have some existing and suitable data somewhere in some form, **it’s usually cheaper / faster to just
  import it with COPY**, even if having to run through some “sed” pipe for smaller adjustments, than to generate it from zero. Especially
  if a lot of random numbers are involved.

* In some rare cases one can speed up the above listed optimization approaches a bit with the **pgbench server-side
  generate** (capital “G” for \-\-init-steps) option. Worth trying for very slow or long-distance networks between the client
  and the DB server.

* If you’re pushing a lot of data into your personal workstation or have access to the actual server, **always prefer feeding
  data over Unix Sockets** vs the TCP stack - it’s basically just a small free lunch! Luckily when you operate under the Postgres
  user, you get that automatically on most distros, but better to verify. And I think it even works on Windows nowadays 😵‍💫

* **The parallel level stuff only makes sense if you’re bogged down by CPU, not IO**. Which commonly is not the case with databases.
  So generally there’s no need to go parallel at all. This is especially the case with typical low and mid-tier cloud
  databases - they get throttled real fast on IO, so no point to reach for some advanced sorcery - just wait it out and have a cuppa.

* **For ultra large datasets I sometimes also bother to disable the Autovacuum on the table level before data generation**, especially if on sluggish storage.
  It will otherwise kick in right after init and make first testing queries unpredictable and slower.

* Probably worth mentioning also that the displayed “pgbench” approaches don’t necessarily only need to apply to “dummy”
  data - especially **the multi-worker chunked approach can also be employed for generating “near to real” business domain data and
  stress test workloads!** One “just” needs to know the characteristics of your target data (I usually peek into pg_stats),
  and some pgbench scripting / functions “fu” :) Hope to demo that some day also.
