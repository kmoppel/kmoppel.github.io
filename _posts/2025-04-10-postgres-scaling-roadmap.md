---
layout: post
title: A roadmap to scaling Postgres
cover-img: /assets/img/scaling.jpg
tags: [postgres, design, performance]
---

After meeting with another database fellow recently - the evergreen topic of database scaling popped up again. That won’t go out of fashion anytime soon I guess, despite the AI advancements and the rise of
Postgres-aware chatbots a la Xata Agent. Namely - when to grab for which designs, technical means or plain hacks.

Anyways, then I remembered that some years ago I wrote a fairly popular (made it to the Hacker News front page even) [blog post](https://www.cybertec-postgresql.com/en/postgres-scaling-advice-for-2021/) on the
same topic - meaning a lot of people actually care, and would not rather throw insane amounts of money to hide problems. But my advice surely needs updating as 4 “Internet years” have changed things a bit.

So here again my attempt on maybe saving some mosquitoes from cannonballs - a sort of ordered guide or starting list, according to my personal “world view” of course, after 20K+ hours in the Postgres trenches.


# Basic Postgres internals knowledge

For long term happiness there’s no way around it really, as far as my experience goes. Do words like  MVCC, autovacuum, WAL, bloat, backend and shared buffers ring a bell? If not then back to school, hush-hush.

# Data modeling

The number one thing to get right of course is not super Postgres specific really - knowing your business use case and scaling needs well. If it’s gonna be a low traffic, non-critical thing - do whatever makes the app developer happy here. But if you’ve somehow “made it”, and find yourself in firefighting mode too often - stop all the Postgres nitty-gritty for the moment and focus on the data flow. Eliminate waste and optimize for efficient Postgres row handling (see the previous point) - disk / cache locality, batching, separating frequently changing data / columns from the rest etc. The sooner and harder migration decisions are made (be it at the cost of a small feature freeze), the smoother the long run upscaling experience.

# Indexing


Just the right amount is the target here. No front-up shotguns. If some “shotgunning” is needed - then preferably with cheap index types if possible. What are those you might ask? Partial indexes, BRIN, Bloom and also GiN for some low-cardinality columns as it beats B-tree deduplication still.


# Hardware++


Sometimes, if growth is linear / foreseeable i.e. not on the internet scale - just do it! This is where in the past I've fallen into the micro-optimization rabbit-holes frequently instead, and delayed “dragging the slider”. But with years - one learns to calculate and look at the TCO more, and sometimes it’s really worth it to keep the design or configuration simple.   


And also in the hardware section - **this might be the biggest change of mentality for me compared to 4 years** - I sometimes actually even recommend a fully managed DBaaS product with some serverless or auto-scaling features to begin with! Makes a lot of sense especially for batch processing / data warehouse type workloads - which is one of the few cloud use cases which actually helps to save money (if that’s a decision parameter even of course). In general fully managed DBaaS are still my personal 3rd choice due to various reasons - behind self-managed and semi-managed (i.e. some K8s operator) options.


# Increased statistics and custom statistics


Worth a look especially if doing heavier, more complex queries. Postgres defaults are super conservative and from a time long ago. Even when you have a 100M rows table, Postgres by default only looks at max 30K rows - meaning planner assumptions are made based on looking at ~0.03% of data, which is representative only of steady and uniform workloads.


Usefulness / gains of custom statistics has been somewhat debatable in the community - but the expressions part (v14+) is definitely useful I’d say, compared to creating some light-weight indexes just to get the statistics to help the planner, as one had to do in the old days.


# Configuration tuning


Here I mean the stuff outside of “shared_buffers” and a few other key parameters, spewn out by all the online config / tuning generators - according to some hardware specs and main access pattern type, e.g. OLTP.

In the old days I think I spent too much time here, instead of “dragging the HW slider”. As all in all, especially with reading, it’s pretty darn hard to do much pure config level magic here if cache rates are insufficient or the data model sucks. With writing a bit more can be done, with the biggest wins of course coming with taking some risks - like the possibility of losing some last commits on crashing.


# Partitioning


Hot-cold data separation is the main prize here. And the holy grail for Postgres (if the business flow allows or can be “bended” a bit of course) - dropping or archiving of complete old partitions, instead of deleting data - which still induces autovacuum and increases disk fragmentation among other things. Just having partions for normal flows is still good as well, as it helps to parallelize autovacuum and reduce long-term bloatedness.


# Compression

Disk-level compression for read-tilted “Big Data” use cases is a no-brainer and pays off heavily usually. Given of course you’re self-managing and actually can do it. Or some 3rd party extension like [TimescaleDB](https://github.com/timescale/timescaledb) can do the trick also mostly, comes with some “but”-s though, as always. See here for an example [hack](https://kmoppel.github.io/2024-08-19-a-timescaledb-analytics-trick/).


# Pre-aggregation

With VIEW-s hiding it from apps / end users would be especially nice here. Usually boils down to UNION-s merging old, already aggregated data and doing it on-the-fly for the latest data. Doesn’t play well with ORM-s though, so sometimes some incremental triggers or an extension like [pg_ivm](https://github.com/sraoss/pg_ivm) is needed. But then again - you need to make sure some stuff is running in the background and apps / users are aware of some things, so still a pretty "late" preference.

# Read replicas


One could say - read replicas are good, but primary-only is even better! Of course it’s desirable if the application CAN support replicas - but on most occasions I see most load going to the primary still, due to routing and transaction dependency problems. And it’s not a free lunch by far - the hardware cost, cluster management, the occasional replay or apply lag causing weird “where’s my row, dude” symptoms - so in short I’d recommend just spending more money on the primary nowadays if you’re not a global webscaler. The good thing here is though that the load balancing and routing problem is largely solved by now - be it either via native [LibPQ](https://www.postgresql.org/docs/17/libpq-connect.html#LIBPQ-CONNECT-LOAD-BALANCE-HOSTS), [pgpool2](https://www.pgpool.net/docs/latest/en/html/runtime-config-load-balancing.html) or [PgCat](https://github.com/postgresml/pgcat).


# Logical Replication replicas


From the physics point of view, feeding LR-replicas is slower than Streaming Replicas. But when used in a targeted way, reducing the synced dataset and with a different set of indexes maybe, one can actually win quite a bit! Also the apply lag is not a topic anymore! Remember - streaming read replicas only work well if the queries are OLTP-style / fast!


# Alternative / hybrid storage engines

Ok this is where we wander out of the pure Postgres territory and things get interesting -  and be warned, you can easily shoot yourself in the foot. In this category fall extensions interfacing for example with the DuckDB engine, or some other more Postgres-y columnar engines like [Hydra](https://github.com/hydradatabase/columnar).


# External caching 


Caching the Postgres data files is good, caching the whole resultset is even better. At the same time you run into the problem of staleness and cache management. Redis / Valkey, Memcached are the usual suspects here, but something new and interesting like [gatewayd](https://gatewayd.io/) could work also for some use cases (where all mutations are guaranteed to happen via the proxy).


# Sharding


Sharding is rarely easy (as tooling is pretty non-existent to start with) - better have a long term view on things here, and test heavily - with all kinds of weird queries, also such not containing the partition key and hammering it hard from hundreds of sessions.


I’d prefer again plain Postgres if possible, i.e. application level sharding, plus maybe something like [PgDog](https://github.com/pgdogdev/pgdog) to keep the app dumb. But [Citus](https://github.com/citusdata/citus) works well enough also - just that it’s my 2nd choice not being “core”, plus adding a fair amount of internal complexity (~1K GH issues) ...


# Postgres derivatives / protocol compatible products


Well, it’s last for a reason in my book…when you’re still facing a wall after all the previous points - maybe you’re solving the wrong problem with Postgres? Losing community support and battle-hardened testing of millions of applications around the world - just very hard for newcomers to get to that level of market penetration.  Talking about products - [OrioleDB](https://github.com/orioledb/orioledb) is the first one that comes to mind for example, but there are others like [YugabyteDB](https://github.com/yugabyte/yugabyte-db) etc.


# Did I miss something?

Let me know in the comments.

Hope it made you think, and feel free to [reach out](https://kmoppel.github.io/aboutme/) for my consulting services when need to scale, or at least want to be ready for it.
