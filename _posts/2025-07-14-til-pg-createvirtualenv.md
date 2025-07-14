---
layout: post
title: TIL - Wow, Debian comes with a pg_createvirtualenv wrapper
cover-img: /assets/img/til_psql_read_only.jpeg
tags: [postgres, psql, testing, debian, ubuntu]
---

After some month of friendly "normal operation" with Postgres, it managed (well, not directly the official Postgres project) to surprise
me during my daily muscle-memory operations again, prompting even one relatively lazy human to actually write about it.

So in short, to my delight - minding some other business, I double-tapped "Tab" on "pg_v", looking for [`pg_verifybackup`](https://www.postgresql.org/docs/current/app-pgverifybackup.html)
(a tool used to check the integrity of backups / FS snapshots taken using `pg_basebackup`) ... and suddenly, something
named `pg_virtualenv` popped up! Wow, what is this I thought? Ok, let's see - [`man pg_virtualenv`](https://manpages.ubuntu.com/manpages/focal/en/man1/pg_virtualenv.1.html)
(`man` btw, is an ancient Linux command pre-dating LLM-s to read up on how some utilities or syscalls work) ... mind blown! 🤯

# What does `pg_createvirtualenv` do? 

In short it's an instant throwaway Postgres instance at your fingertips, for those more temporary tasks.

# What did I use before?

So how did I spin up quick temp instances in the past? By either:

* Using some 3rd party utilities
  - Like for example [ephemeralpg](https://github.com/eradman/ephemeralpg) or another similar more
    popular project I fail to recall currently ...
* Docker
  - Especially for really old Postgres versions
  - Or when you need exact minor versions for some reason
* Plain "initdb" + "pg_ctl start" on the CLI
  - For those more or less fresh versions of Postgres that I have installed, with a few common parameters set. In short
    something like this:
```commandline
/usr/lib/postgresql/15/bin/pg_ctl  -D /tmp/pg15 -l /tmp/pg15/logfile -o "-p 7432 --unix-socket-directories='/tmp' --shared_preload_libraries='pg_stat_statements'" start
```

# A quick howto on pg_createvirtualenv

Couldn't get much easier really ...

```
pg_virtualenv -v 15 psql
Creating new PostgreSQL cluster 15/regress ...
psql (17.5 (Ubuntu 17.5-1.pgdg24.04+1), server 15.13 (Ubuntu 15.13-1.pgdg24.04+1))
Type "help" for help.

postgres=#
```

And in the background we get an instance in the `/tmp` folder, running under our current `$USER`:
```
$ ps aux | grep regress
krl       105362  0.0  0.0 226532 32112 ?        Ss   16:31   0:00 /usr/lib/postgresql/15/bin/postgres -D /tmp/pg_virtualenv.eUUQkX/data/15/regress -c config_file=/tmp/pg_virtualenv.eUUQkX/postgresql/15/regress/postgresql.conf
krl       105363  0.0  0.0 226668  6160 ?        Ss   16:31   0:00 postgres: 15/regress: checkpointer 
krl       105364  0.0  0.0 226680  6704 ?        Ss   16:31   0:00 postgres: 15/regress: background writer 
krl       105366  0.0  0.0 226532 10184 ?        Ss   16:31   0:00 postgres: 15/regress: walwriter 
krl       105367  0.0  0.0 228160  8580 ?        Ss   16:31   0:00 postgres: 15/regress: autovacuum launcher 
krl       105368  0.0  0.0 228108  7680 ?        Ss   16:31   0:00 postgres: 15/regress: logical replication launcher 
krl       105390  0.0  0.0 229036 13728 ?        Ss   16:31   0:00 postgres: 15/regress: krl postgres 127.0.0.1(54116) idle
```

And with some "tuning" performed - most notably `fsync=off`.

```
postres=# SELECT ... FROM pg_settings WHERE boot_val IS DISTINCT FROM reset_val ;

 category_shortened │               name               │ current_setting │     pg_default      │ unit │       source       
────────────────────┼──────────────────────────────────┼─────────────────┼─────────────────────┼──────┼────────────────────
 Connections        │ port                             │ 5435            │ 5432                │ ¤    │ configuration file
 Connections        │ unix_socket_directories          │ /tmp            │ /var/run/postgresql │ ¤    │ configuration file
 Preset             │ lc_collate                       │ en_US.UTF-8     │ C                   │ ¤    │ default
 Preset             │ lc_ctype                         │ en_US.UTF-8     │ C                   │ ¤    │ default
 Preset             │ server_encoding                  │ UTF8            │ SQL_ASCII           │ ¤    │ default
 Preset             │ shared_memory_size               │ 143MB           │ 0                   │ MB   │ default
 Preset             │ shared_memory_size_in_huge_pages │ 72              │ -1                  │ ¤    │ default
 Write-Ahead        │ fsync                            │ off             │ on                  │ ¤    │ configuration file
 Write-Ahead        │ wal_buffers                      │ 4MB             │ -1                  │ 8kB  │ default
(9 rows)

```

## Is it perfect?

Well, if ones asks like that, then obviously not :)

Some light "gripes":

* The `pg_lsclusters` wrapper is not showing the temp instances
* The default port numbers are "too close" to the permanent instances I think.
  - Just +1 added to the last used port (i.e. 5432 or 5433 usually) - I'd prefer something from another "namespace"
* And well if to already start complaining - when `fsync=off`, there's little point of having `full_page_writes=on` as well...
* 4MB `wal_buffers` is from the last decade as well
* `wal_level` could be minimal as well to save on those data loading bytes
* As with default Postgres, the `pg_stat_statements` extension is not enabled by default!
  - So if planning to do some query performance troubleshooting as well, you're better off with something like `pg_virtualenv -v 15 -o port=6666 -o shared_preload_libraries='pg_stat_statements' psql`

*PS* The bigges footgun - if you accidentally exit from your `psql` session - it's "sayonara" :) 

# The Postgres ecosystem is a beast

It's a full time job I guess to keep track of everything by now. Out of curiosity I even did a "git blame" for `pg_virtualenv`
...and seems it has been there for ages! Really weird that I ran into it just now, given that I've tried a few "competing products" in the past as well.

But yeah - when working like me, with a very fragmented customer base, running Postgres in weird and ancient configurations
is needed quite often - wether to check some planner behaviour or some internal metrics / monitoring views - it comes in handy!
Hope for you as well.

*PS - feel free to [contact](https://kmoppel.github.io/aboutme/) me if need a bit of help with Postgres - I've put in my
20K+ hours with many Postgres-heavy top tier businesses in Europe so that you don't have to re-invent the wheel*
