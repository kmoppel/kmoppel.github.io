---
layout: post
title: 5 years in PostgreSQL major versions performance - anything surprising?
tags: [postgres, performance, beta]
---
First off - :wave: heya old friends, it’s been a while! After a slight "busy-with-other-things" hiatus (changing jobs, genome
multiplication) I thought it’s about time to hit the keyboard again to avoid becoming overly “rusty” with writing and
also to help my ageing brain in future as it’s pretty neat actually to google your own past articles if you’ve forgotten
some of the nitty-gritty, thus could theoretically even be a net positive “investment” in the end.

## v15 ahead, captain! And v10 almost overboard

Anyways, as very soon it will again be a party of sorts for all Postgres friends in the form of a new major version
release - v15 - I caught myself thinking about any possible performance wins again (hmm, is there something wrong with
me?)...and as it’s kind of a mini anniversary this time: 5 years since the introduction of the new rounded-to-full-numbers
version naming scheme!

Thus I thought it would be cool to do some “archeology” and look a bit past the usual "verX vs verX+1"
comparison target and start from all the way from v10! That should definitely be interesting as there have been
_A LOT_ of changes in that time span - ca 2000 items of  “release-note-worthy” stuff to be exact…mind boggling! But yeah
sadly the release notes and documentation in general are very skimpy about some performance gains (well on the hackers
mailing one could find something) and probably for a good reason even - there are surely a gazillion different use case,
configuration tuning and hardware combinations out there, so that blanket statements will probably look pretty ridiculous
in a very short time - remember, 640KB of RAM should be enough for anyone, right :wink:.

So what to do about this vagueness with performance numbers? Right, as you might have guessed - time to roll up one’s
sleeves and put old and new elephants up for some trials! So please read on for some details on my simple test setup or
just jump to the bottom for some of my findings.

A heads up - note that I didn’t want to test anything approaching “big data”, but rather “in-mem”, so if you juggle huge
amounts of data and hit the disk mostly then I guess the whole exercise might seem pretty pointless. But it should nevertheless
show something about possible changes in how Postgres creates execution plans, does index and full scans, unpacks tuples
and does a bit of number crunching.

## Test setup

Some things don’t change in a year - my weapon of choice for throwing together some “quick and dirty / personal usage”
style tests involving Postgres, is still the great Swiss Army knife “pgbench”, albeit this time with some custom SQL
mixed into the pot as the default OLTP-style test scenario is a bit one-sided and doesn’t bring out the fact that
Postgres can also be used successfully for more analytical use cases (up to a certain limit though - it doesn’t employ
columnar storage or too aggressive compression by default). So I added a few JOIN and “mass data” type of queries that
seemed “common”-ish to me at the time, but not giving it too much though honestly as it’s anyways very hard to get some
definitive results, so keep that in mind. 

### Test queries

```sql
-- Standard "tcp-b"-like --skip-some-updates 
UPDATE pgbench_accounts SET abalance = abalance + $1 WHERE aid = $2;
SELECT abalance FROM pgbench_accounts WHERE aid = $1;

-- Full scan  
SELECT count(*), avg(abalance) FROM pgbench_accounts;

-- Assumes a 4x reduced (aid % 4 = 0) clone of pgbench_accounts
SELECT count(*) FROM pgbench_accounts JOIN pgbench_accounts_reduced USING (aid);

-- Top 5 accounts for each branch - assumes a new index on pgbench_accounts(bid)
SELECT bid, abalance FROM pgbench_branches b JOIN LATERAL (SELECT abalance FROM pgbench_accounts WHERE bid = b.bid ORDER BY abalance DESC LIMIT 5) a ON true;
```

### Hardware

5x GCP e2-standard-4 VMs with 4 vCPU and 16GB of RAM,  50GB Standard persistent disks (very low IOPS)

### Software

**OS**: Debian 11 (Bullseye)

**Postgres**: v10 to v15 with latest minor versions and Beta 2 for v15 (not the latest anymore as of writing). Also some
basic “best practice” config tuning applied (25% RAM shared_buffers, random_page_cost=1.25, max_parallel_workers_per_gather=1) given the hardware.

**Working set size**: 3 sizes for each query - fits shared buffers, fits RAM, slightly over RAM so some light access for
slow disks that should attenuate the possible algorithmical disk access improvements well on Postgres side.

**Test runtime**: 1h for each “working set size” / scale, query protocol (plain vs extended), query, PG version combination

**Level of parallelism**: I chose a fixed “active clients” count of 2 to not max out the CPU and avoid some context switching randomness.

**Measuring method**: Postgres built-in “pg_stat_statements” extension stats aggregation

Also this time in contrast to the “old times” you might have noticed that instead of real hardware I decided to run the
tests on cloud VMs…as seems it’s becoming a norm also for the DB world slowly, or at least things are moving there.

Again on the chosen dataset sizes - yes, it’s all almost “in-mem” with a small dataset, so you might think it’s a bit alien to real life use cases - but remember that cloud storage latencies / randomness are much higher than for local disks, making it hard to actually benchmark the software. Also in theory (if you have deep pockets) one can easily “click-click” yourself multiple terabytes of RAM now on AWS for example, giving you a very similar “mostly in memory” feel even for Big Data.

## High level results

After pressing “Enter” it took 8 days before I could finally rub my hands together from excitement…but to my horror the
initial joy quickly turned into almost despair -as it became apparent there was a LOT of data (900 rows) to be analysed,
much more than I had anticipated - combinatorics can sure be surprising sometimes :sweat_smile: So after an hour poking at the data and comparing individual rows I could finally formulate what I was actually looking for and could write some analytical queries using Window Functions (PS if you don’t command them yet definitely go and [learn them](https://www.postgresql.org/docs/current/tutorial-window.html) - they’re one of the best parts of SQL, seriously). So without further ado, here some comparative numbers below.

Note that in the end for simplicity I actually decided to just ignore the “in between” Postgres versions and look only at v10 and v15 for now as it seemed a bit too much work to actually summarise it nicely…hope to tackle it in the future though and you have the full data [here](https://github.com/kmoppel/pg-perf-test-v10-v15/blob/main/pgss_results_dump.sql), so feel free to knock yourself out there or do some fact-check for below numbers if you have the time.

| Query                                                | Mean exec time for v15 (ms) | Exec time change (%) | Stddev change (%) |
|:-----------------------------------------------------|:--------------------|:-----------------|:-|
| SELECT abalance FROM pgbench_accounts WHERE aid = $1 | 0.01 | 6.72 | 5.24 |
| UPDATE pgbench_accounts SET abalance = abalance + $1 WHERE aid = $2 | 7.17 | 18.9 | 5.0 |
| SELECT bid, abalance FROM pgbench_branches b JOIN LATERAL (SELECT abalance FROM pgbench_accounts WHERE bid = b.bid ORDER BY abalance DESC LIMIT $1) a ON $2 | 465139.5 |-18.8 | 279.4 |
| SELECT count(*) FROM pgbench_accounts JOIN pgbench_accounts_reduced USING (aid) | 34582.4 | 123.6 | 302.1 |
| SELECT count(*), avg(abalance) FROM pgbench_accounts | 454882.0 | -21.9 | 342.1 |


## Two anomalies

### Slow JOIN on pgbench_accounts_reduced

The first thing that I noticed when looking at the query results was of course the 123% slowdown on the JOIN query - must have
been something with my test I thought! But after a manual re-check - indeed, Postgres v15 beta picked a worse plan than
on v10! These things happen from time to time of course and can usually be worked around - but from 3 custom queries I
chose to already hit one…was a bit surprised, good time to buy some lottery tickets probably :smile:

But don’t worry too much - based on my 10 years working daily with Postgres I can attest that it is something rare enough
and again - it’s on a beta version.

From the technical side by the way - after running through an “EXPLAIN ANALYZE” I saw that JIT had kicked in, so I disabled
that and thought that must have been it...but still got a slower plan with a HASH join selected instead of a MERGE join!
Though it did have a lower cost estimate than the v10 MERGE plan (which is by the way usually a rare sight and kind of a
"fallback" JOIN type), so technically Postgres did what it had to do. Only after disabling hash joins I could see that v15
was indeed by around 20% faster, which is pretty nice actually!

### Deterioration in standard deviation for the analytical queries

Hmm, quite some large 200+% changes in stddev…so just in case re-ran queries on both versions again:

```sql
select count(*), avg(abalance) from pgbench_accounts
```

Hhmm both versions had exactly the same "Parallel Seq Scan" plan, as it’s a very simple query - and weirdly enough, v15 was
constantly faster…but just more jumpy somehow. Nothing obvious came to mind - maybe some "cloud" / "noisy neighbour" effect"? 

```sql
SELECT count(*) FROM pgbench_accounts JOIN pgbench_accounts_reduced USING (aid)
```

Here a bit more understandble as the same query explained in the previous “Slow JOIN” paragraph - JIT kicking in
unneccesarily + a slightly slower plan was selected.

```sql
SELECT bid, abalance FROM pgbench_branches b JOIN LATERAL (SELECT abalance FROM pgbench_accounts WHERE bid = b.bid ORDER BY abalance DESC LIMIT $1) a ON
```
Again seems JIT kicked in too early…but once disabled on v15 it normalized and actually pulled ahead v10 in runtime by a decent 40%!


## Trying to conclude something

Ok phew, now to the hard part - so what did I actually learn from this exercise, if any? Some thoughts:

* Remember that I tested a **v15 Beta 2** release (Beta 3 was sadly release soon after I started my testing)
* After excluding the query with the above detailed planner anomaly from test results **there was a speedup of only 4% over all queries and scales**. Not much? Yes. Surprising? No - pretty much logical actually as PostgreSQL has been rock solid and (mostly) well optimized the whole decade I’ve been working with it! Also it was a very static, mostly read-only and almost “in-memory” test setup where many recent major optimizations (index de-duplication, bottom up index cleanups etc) couldn’t surface.
* Postgres planner is not the most complex one out there - it can still misjudge things, so **as ever, it’s essential to know your way around EXPLAIN plan, planner constants and certain constructs** (like LATERAL) that nudge Postgres towards a plan of your liking.
* Query protocol (plain SQL vs prepared statements) still plays a huge role for smaller / faster queries - the 2 non-analytical **simple indexed-access queries got a 20% and 50% boost just by using prepared statements!** Nothing new here of course , but just to repeat :)
* **Seems cloud VMs cannot be considered too trustworthy for such relatively short-running tests** - the statistical Coefficient of Variation (~7%) over all queries / scales over different 5 hosts was actually higher than the measured Postgres speedup of 4% (excluding the outlier query)...so next time I’ll still spin up some real hardware again.
* **All in all a very decent performance from good old v10**, which will actually be EOL-ed already in a few months! So those still running on that in a safe network (no security updates, remember!) and don’t want to take that downtime to upgrade, don't have to worry too much about performance part at least.
* Keep in mind that testing performance is hard and there are a ton of things I didn’t test - partitioning, heavily concurrent / multi-user access, long term degradation due to bloat. So that the custom SQL queries that I conjured up seem now pretty one-sided (more on the analytical side) to me, so maybe **take it just as “something”**.

By the way the scripts I used can be found [here](https://github.com/kmoppel/pg-perf-test-v10-v15) if you want to try out something similar. Additionally you’d only have to create separately a Postgres “results db” accessible from test VMs, to push the results into a single table for easy analysing via SQL. Also if you see some obvious errors in my logic or have other thoughts, would be great to get a ping in the comments section.

That’s it for this time! Hope to return back with a similar piece when v15 final is released - take good care of your databases and other pets in the meantime :elephant: :dog:

PS Special thanks to my employer Cognite for being blogging-friendly and allowing me to burn those CPU-s for a week! By the way, we’re also hiring for a few Database Engineer positions on various engines, so why not to [take a look](https://www.cognite.com/company/careers) when don't mind having me as a colleague :wink:
