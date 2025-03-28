---
layout: post
title: TIL - Starting in read-only mode the easy way
cover-img: /assets/img/til_psql_read_only.jpeg
tags: [postgres, psql, libpq]
---

Once a year or so Postgres manages to surprise me for some of those daily DB micro-operations, that come from muscle
memory usually, without any braincells spent. Today to my delight I stumbled on one such again - for the task of
starting a `psql` session in a safe read-only mode.

As it happens - even after long years with Postgres, sometimes I prefer not to troubleshoot or investigate some more
critical instances with almighty and possibly destructive superuser powers.

So how did I achieve this previously? By either:

* Doing an ad-hoc `SET SESSION` when need arose (the common case)
* Switching to a secondary `psql` startup config file, executing the former automatically at session start
* Create a special read-only user and only log on using that

# Before the enlightenment

## Setting the session characteristic via SQL

Most of the time when firing up `psql`, I'm actually on my workstation, just experimenting on something etc. Meaning I
usually create and destroy objects at will, and don't want to be limited in any way. So that when I actually do connect
to some more  important DB, I just execute the below directly after connecting:

```sql
SET SESSION CHARACTERISTICS AS TRANSACTION READ ONLY ;
```

which disables any accidental changes, and one can copy-paste stuff more stress-free into the console, which - hush -
nowadays can on rare occasions even be created by our robot-friends.

And when it appears I actually need to do some changes, I'd run:

```
BEGIN TRANSACTION READ WRITE ;
...
COMMIT ;
```

## Using a separate `.psqlrc` config

The first, ad-hoc *SET SESSION* approach can sometimes get a bit annoying though in the end, if need to re-connect a
lot into customer systems during a day.

Then I switch to a config file based approach!

In case you weren't aware - **one can create startup config files for *psql*!** Which can contain basically any normal SQL,
or *psql* meta-commands, like setting the pager, auto-commit mode, etc - and is then executed line-by-line on session start.
And a bit scarily actually - it's done automatically on *psql* session start when you have a `~/.psqlrc` file defined, or
in any other [default location](https://www.postgresql.org/docs/17/app-psql.html#APP-PSQL-FILES-PSQLRC) for that matter.

This haha, reminds me of a mean prank I once pulled on a new junior colleague. Namely - on a shared Postgres server that we both administered,
I suddenly planted a rigged *.psqlrc* file to auto-execute something like:

```
\pset tuples_only on
SELECT 'WARNING: Instance shutdown triggered, rebooting in 3 seconds ...' ;
SELECT 'Press CTRL-ALT-9 + CTRL-ALT-5 to abort' ;
SELECT pg_sleep(3);
SELECT 'Just joking ;)' ;
\pset tuples_only off
```

and then watched him across the desk suddenly starting to hit the keyboard frantically :D Fun times. But the lesson is actually
valid - **whenever on a shared / untrusted system, it's always good to disable the default .psqlrc processing** via the
`-X / --no-psqlrc` flag!

Anyways, in short I have a second, slightly modified, copy of my `.psqlrc` file, with `SET SESSION CHARACTERISTICS AS TRANSACTION READ ONLY ;`
executed as first thing, and if working more extensively with a very important system I activate it via:

```
PSQLRC=~/.psqrlrc_ro psql ...
```

Btw, my `.psqlrc` looks usually something like that: [https://gist.github.com/kmoppel/d27bf1d31affcc26f56fd3a9bd596477](https://gist.github.com/kmoppel/d27bf1d31affcc26f56fd3a9bd596477)


## A special-purpose read-only user 

On some rare occasions, I've actually went as far as to create a temporary "gelded" user - that can only read. This comes
handy when having to use some other query tools or apps to communicate with the database. It goes something like that:

```
CREATE USER kaarel_ro IN ROLE pg_read_all_data, pg_monitor ;
\password kaarel_ro
```

Note here that it's explicitly not a good idea to use the `CREATE USER ... PASSWORD 'password'` form here, as that can
leak plain-text passwords into the server log!

# The enlightenment

Long story short - today I stumbled on a better, more universal way to start in read-only mode 🎉 By setting the
`default_transaction_read_only` Postgres parameter on session start! À la:

```
$ PGOPTIONS='-c default_transaction_read_only=on' psql -X
psql (17.4 (Ubuntu 17.4-1.pgdg22.04+2), server 16.8 (Ubuntu 16.8-1.pgdg22.04+1))
Type "help" for help.

postgres=# create table t1();
ERROR:  cannot execute CREATE TABLE in a read-only transaction
```

And why the joy?

Firstly - one doesn't have to rely on *.psqlrc* for automatic read-only mode, meaning one can disable it with the *-X* flag if
needed (remember, can be sort of unsafe on shared systems as shown above). And one doesn't need a separate "read only"
*.psqlrc* config file anymore.

And secondly - although 95% of time I rely on good ol' *psql* (backed up by *DBeaver* in case some data needs to
be fixed as well) the nice thing about this approach is that its client agnostic! As setting startup
parameters is actually a driver level feature. So one could as well set it from a JDBC connect string:

```
jdbc:postgresql://localhost:5432/mydatabase?user=myuser&password=mypassword&options=-c%20default_transaction_read_only%3Don
```

Hope it comes in handy for you as well.

*PS - feel free to [contact](https://kmoppel.github.io/aboutme/) me if need a bit of help with Postgres - I've put in my
20K+ hours with many Postgres-heavy top tier businesses in Europe so that you don't have to re-invent the wheel*
