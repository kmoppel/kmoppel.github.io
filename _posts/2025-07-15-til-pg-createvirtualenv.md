---
layout: post
title: TIL - Debian comes with a pg_virtualenv wrapper!
cover-img: /assets/img/pg_createvirtualenv.jpg
tags: [postgres, psql, testing, debian, ubuntu]
---

Some weeks ago Postgres (well, not directly the official Postgres project) again managed to surprise me during my daily
muscle-memory operations, prompting even one relatively lazy human to actually write about it.

So while minding some other common Postgres business, I double-tapped "Tab" to complete on "pg_v", looking for [`pg_verifybackup`](https://www.postgresql.org/docs/current/app-pgverifybackup.html)
(a tool used to check the integrity of backups / FS snapshots taken using [`pg_basebackup`](https://www.postgresql.org/docs/current/app-pgbasebackup.html)) ...
and suddenly, something named [`pg_virtualenv`](https://manpages.ubuntu.com/manpages/focal/man1/pg_virtualenv.1.html) popped up!

Wow, what's that I thought? Ok, let's see - typed in `man pg_virtualenv` (`man` btw, is an ancient Linux command pre-dating
ChatGPT & Co, to read up on how some utilities or syscalls work) ... and mind blown! 🤯

# Ok so what does it provide? 

In short - it's an instant throwaway Postgres shell at your fingertips, for those more temporary tasks. With an almost
normal Postgres instance behind it as well of course, for app usage etc.

# What did I use before?

So how did I spin up quick temp instances in the past? By either:

* Using some 3rd party utilities
  - Like for example [ephemeralpg](https://github.com/eradman/ephemeralpg) or another similar more
    popular project I fail to recall currently ...
* Docker
  - Especially convenient when you need exact minor versions for some reason, not to compile your own.
* Plain "initdb" + "pg_ctl start" on the CLI
  - Recently my main "goto" actually for those more or less fresh versions of Postgres that I have installed. With a few
    common parameters set it looks something like this:

```commandline
/usr/lib/postgresql/15/bin/initdb -D /tmp/pg15 && \
/usr/lib/postgresql/15/bin/pg_ctl -D /tmp/pg15 -l /tmp/pg15/logfile -o "-p 7432 --unix-socket-directories='/tmp' --shared_preload_libraries='pg_stat_statements'" start
```
Not exactly complex or awful as well, but one has to also mind the cleanup phase when done with larger instances... 

# A quick howto on pg_virtualenv

Couldn't get much easier really ... for when I need to check if some monitoring query works with Postgres 15: 

```
pg_virtualenv -v 15 psql
Creating new PostgreSQL cluster 15/regress ...
psql (17.5 (Ubuntu 17.5-1.pgdg24.04+1), server 15.13 (Ubuntu 15.13-1.pgdg24.04+1))
Type "help" for help.

postgres=#
```

## What's in the box

In the background we get an instance in the `/tmp` folder, running under our current `$USER`:

```
$ ps -efH | ag regress
krl      3312253 3275277  0 00:55 pts/15   00:00:00           ag -ifaz --silent regres
krl      3312174    7183  0 00:55 ?        00:00:00     /usr/lib/postgresql/15/bin/postgres -D /tmp/pg_virtualenv.vz1F1s/data/15/regress -c config_file=/tmp/pg_virtualenv.vz1F1s/postgresql/15/regress/postgresql.conf
krl      3312175 3312174  0 00:55 ?        00:00:00       postgres: 15/regress: checkpointer 
krl      3312176 3312174  0 00:55 ?        00:00:00       postgres: 15/regress: background writer 
krl      3312178 3312174  0 00:55 ?        00:00:00       postgres: 15/regress: walwriter 
krl      3312179 3312174  0 00:55 ?        00:00:00       postgres: 15/regress: autovacuum launcher 
krl      3312180 3312174  0 00:55 ?        00:00:00       postgres: 15/regress: logical replication launcher 
krl      3312219 3312174  0 00:55 ?        00:00:00       postgres: 15/regress: krl postgres 127.0.0.1(43786) idle
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

Well, if ones asks like that already :)

Some light "gripes":

* The `pg_lsclusters` wrapper is not showing the temp instances
  - Fine if not by default, but there really could be some flag at least 
* The default port numbers are "too close" to the permanent instances I think - just +1 added to the last used port (i.e. 5432 or 5433 usually)
  - I'd prefer something from another "namespace", so I specify something like `-o port=6666` as well usually
* When `fsync=off`, there's little point of having `full_page_writes=on` as well...
* Also `wal_level` could be `minimal` to save on those data loading bytes, no one is going to create replicas or take snapshots from a throwaway instance...
* Default server / error log access is a bit inconvenient (possible via `/proc/$postmaster_pid/fd/1`), so that for
  non-trivial use cases you'd add `-o logging_collector=on` as well.

So if planning to load quite a bit of data, and do some query performance troubleshooting as well, you're better off with something like:
```
pg_virtualenv -o shared_preload_libraries='pg_stat_statements' -o full_page_writes=off -o wal_level=minimal -o max_wal_senders=0 -o random_page_cost=1.25 psql
```

*PS* The biggest footgun with `pg_virtualenv` probably is that if you accidentally exit your main `psql` session - it's "sayonara" :)

# The Postgres ecosystem is a beast

It must be a full time job I guess nowadays, to keep track of everything in the Postgres space...out of curiosity I even checked
the history for `pg_virtualenv` - and seems it has been there for ages! Really weird that I ran into it just now, given
that I've tried a few "competing products" in the past as well.

In any case - when working with a fragmented customer base like me, having different Postgres versions at your fingertips
comes handy quite often - whether to check some planner behaviour or some internal metrics availability for example.

Hope the tool comes in handy for you as well! Given you're on Ubuntu / Debian of course...

*PS - feel free to [contact](https://kmoppel.github.io/aboutme/) me if need a bit of help with Postgres - I've put in my
20K+ hours with many Postgres-heavy top tier businesses in Europe so that you don't have to re-invent the wheel*
