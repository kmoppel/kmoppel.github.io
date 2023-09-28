---
layout: post
title: TIL - When was my Postgres cluster initialized ?
cover-img: /assets/img/til-when-was-my-postgres-cluster-initialized.png
tags: [postgres, initdb, rds]
---

Today again a nice finding after 12 years with Postgres - seems one can actually retrieve the time when a cluster / instance
was once initialized! Given of course you haven't dump-restored or in-place upgraded the instance in the mean time...

So far on those rare occasions when I've actually had to look up something like that () - I've worked around it (as not reported
by *pg_controldata* also for example) by either:
 
* digging into version control
* just looking at the file system timestamps of a file (hopefully) not touched after the initial init - one such could be the 
  $PGDATA/PG_VERSION file

But now seems there's a better, or at least cooler, way 

# Finding out the cluster init date

## Via SQL

When on v10+ or higher or on a manged instance with no $DATADIR access:

```sql
select to_timestamp ( system_identifier >> 32 ) as cluster_init from pg_control_system();
      cluster_init      
────────────────────────
 2023-09-11 22:27:01+03
(1 row)
```

## Non-SQL, e.g. in Bash

When still below v10 or just have local access:

```bash
SYSID=$(pg_controldata /var/lib/postgresql/16/main/ | grep 'system ident' | grep -oE '[0-9]+')
echo $SYSID 
# 7277652093081719641
date -d @$((SYSID >> 32))
#  11 sept  2023 22:27:01 EEST
```

# How did I stumble on something like that ?

Now the weird part - we had a bunch of instances on RDS reporting the same *system_identifier* 🤔 So that I had to open
up the [source](https://doxygen.postgresql.org/xlog_8c.html#aa67c99e001f4fb4a3e6f5ae99d7efddf) to see how on earth could
that be possible. In short - it shouldn't! In theory the sys IDs should be unique exactly due to this "init time" component
(plus some bits from the init process ID). But seems RDS is copy-pasting some pre-bootstrapped folders around
or just firing up some pre-bootstrapped containers - in any case a flawed concept in my opinion, inhibiting for example
easy counting of unique instances and replica pairs by a "group by system_identifier" on generic (instance unaware) metrics
we harvest.

By the way, the system ID also is checked when replaying WAL - so you should try to definitely avoid this re-use pattern in case you
run self-managed and there's even as much as a slight theoretical risk of mixing up WAL buckets or folders.


> Cover image Stable Diffusion prompt - *blue elephants in a royal dining hall, clocks on the wall, baroque style*
