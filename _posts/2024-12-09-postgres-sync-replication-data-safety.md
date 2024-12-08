---
layout: post
title: Postgres Synchronous Replication - a 99.99% guarantee only
cover-img: /assets/img/weights.jpg
thumbnail-img: /assets/img/weights.jpg
tags: [postgres, replication, ha]
---

At last week's local Postgres user group [meetup](https://pgug.ee/meetings/08/) here in Estonia, one of the topics was HA
and recent Patroni (the most popular cluster manager for Postgres) improvements in supporting quorum commit, which by the
way on its own has been possible to use for years. Things went deep quickly and we learned quite a bit of course. Including
a good reminder that you shouldn't build your bank on Patroni’s default synchronous mode :)

Anyways, during the hallway track (which sometimes are as valuable as the real ones) got an interesting question - with
some 3+ quorum nodes, is Postgres then 100% bulletproof against all kinds failures? Excluding meteorites, rouge DBAs
and such of course. One could think so, right? Nope.

Had already forgotten about that chat, but some serendipity kicked in today and stumbled on a trending Hacker News [article](https://news.ycombinator.com/item?id=42293937)
on the same topic basically...and though, wow this fact could indeed not be so well known and probably worth a post.  

# Synchronous HA doesn't always equal 100% data safety

In short **synchronous HA can be broken** in a few scenarios that I know of (please, please comment if you know of some
more) - in short it could be summarized as: **a\)** fooling around with
["synchronous_commit"](https://www.postgresql.org/docs/current/runtime-config-wal.html#GUC-SYNCHRONOUS-COMMIT) or **b\)** a
combination of too optimistic programming together with some quite normal replica or network failures...which does happen
all around the world every day.

# Asynchronous commit

Asynchronous commit means that the server operator globally, or an user on session / transaction level on-purpose (haven't heard
of accidental cases yet) enables "relaxed" committing via the `synchronous_commit` parameter - meaning: we're not really waiting for
guarantees on `COMMIT`, but are instead handing over the data and saying:
> "please do a commit for me ASAP Postgres, gotta go, love you, bye".

This back-door in Sync HA is thankfully quite well summarized also in Patroni [docs](https://patroni.readthedocs.io/en/latest/replication_modes.html#synchronous-mode).

```
For use cases where losing committed transactions is not permissible you can turn
on Patroni’s “synchronous_mode”. When synchronous_mode is turned on Patroni will
not promote a standby unless it is certain that the standby contains all
transactions that may have returned a successful commit status to client. [2]. 
 
And the reference [2] from the footer:

Clients can change the behavior per transaction using PostgreSQL’s "synchronous_commit"
setting. Transactions with "synchronous_commit" values of off and local may be lost on
fail over, but will not be blocked by replication delays.
```

So one could assume that diligent users are aware of the consequences and **use async commiting only for suitable use
cases** - for example ingesting some metrics or such, that will be re-harvested in a minute or so, and a small data loss
won't change the big picture.

# Sloppy programming

Now to more serious threats. Assuming we have a minimal 2 node Sync HA setup (mind you - the recommended minimum number
of nodes for sync replication is 3!) the scenario looks something like that:

1. A network partition happens or the replica node dies - something pretty common actually, even a question of time
  really when you run dozens of clusters... 
2. Now, although sessions can still technically work and commit, commit ACKs from replicas are not coming in anymore and technically succeeded app
  transaction are not returned to callers - things start to “hang” so to say. Resulting quite often also in “out of connections”
  errors, being only then noticed by the ops team in many cases 🙂
3. The primary is now running out of connections and the ops team makes a hasty decision to "just in case" restart the
  server, or start terminating the longest running active connections to make some “room” for new connections. An easy
  mistake to make actually under pressure.
4. Applications now error out of course and thinking work was not completed (but commits were actually persisted,
  we were just waiting on `wait_event=SyncRep`), do some work again. Some variations how this can manifest:
  - Just duplicate events if constraints are not good enough and the replica comes back soon. Here the transaction hangs
    again of course if the replica is gone, thus mostly a huge amount of work won't be performed.
  - On re-connect the app recognizes that the previous change is there via a WHERE lookup or a unique constraint
    violation - which doesn't hang in that case! - and is usually read as "ok good, data is there, now let's propagate the fact somewhere"...but then also
    the primary crashes and burns 🔥, and a replica is eventually promoted...BUT of course the
    propagated event data is nowhere to be found anymore in the source system...

# The sneaky pg_cancel_backend() function

And an especially sneaky situation to finish up - with replica gone, sessions pile up again, and some power users or
DBAs (junior) start killing older sessions, using the `pg_cancel_backend()` function...Which is quite common
actually, as by default all users can cancel their own queries from other sessions using that function, whereas the more
powerful `pg_terminate_backend()` is for superusers or `pg_signal_backend` group members only. But a little
known **nasty side aspect of `pg_cancel_backend()` is that sessions in `SyncRep` wait state actually report success to the caller**!

```
krl@postgres=# insert into t1 select 1 ;
WARNING:  canceling wait for synchronous replication due to user request
DETAIL:  The transaction has already committed locally, but might not have been replicated to the standby.
INSERT 0 1
```

This, followed by a primary burning down and replica promoted can of course lead to all kinds of nasty ghost data propagation
issues, **when the application does not check for those extra warning messages!** Which, is not an overly common practice
based on my experience. Maybe as some drivers, like `psycopg` for example also have moved the warning check a bit too much out of
"sight", i.e. not attached to the commonly used cursor object.

# Recommendations

* If you really need 100% correctness, i.e. no "ghost data" propagated somewhere - you need to do it yourself in app code
  the hard way, via - [2PC](https://www.postgresql.org/docs/current/two-phase.html). Which of course comes with its own
  set of disclaimers and is thus NOT enabled by default at all with Postgres - so better know what you're doing 🙂
* Prefer a "warning messages"-friendly driver like the JDBC one for critical stuff 
* For the class of "weak unique constraints on purpose" (performance or high-concurrency lock reduction!) cases, one could
  also consider liberal use of `txid_current()` before critical sections and `txid_status()` functions to re-check on errors.

Throwing in a personal wishlist item here also - would be very nice if Postgres had some special parameter to time-box
and fail such `SyncRep` waits still, as currently statement and transaction timeouts added in v17 do not actually apply weirdly enough.
Well actually as the transaction did succeed and was also persisted, just missing replica ACKs...so looks like some Catch-22 situation here.

# Summary

Although all the troublemaker examples about handling Sync Replication HA failures can be labelled as developer or DBA
sloppiness, it's still not totally off here to cite the classic from Ben Franklin:
> Nothing is certain except death and taxes

*PS - feel free to [contact](https://kmoppel.github.io/aboutme/) me if need a bit of help with Postgres - I've put in my
20K+ hours with many Postgres-heavy top tier businesses in Europe so that you don't have to re-invent the wheel*
