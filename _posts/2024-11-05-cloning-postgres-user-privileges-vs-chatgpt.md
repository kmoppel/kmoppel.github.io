---
layout: post
title: Cloning Postgres user privileges vs ChatGPT
cover-img: /assets/img/blue_elephant_mannequines.jpg
tags: [postgres, privileges, grants, documentation, cloning, beginner]
---

At [pgConf.eu](https://www.postgresql.eu/events/pgconfeu2024/schedule/) in Athens - by the way, reported to be the largest
PostgreSQL conference to date, attesting to the software’s undeniable success! - I got into a sort of interesting hallway
discussion. I guess my chat companion was relatively new to the ecosystem - and said something along the lines of:

*“Postgres is of course great, but some common tasks can still cause
quite a bit of frustration - like finding out which privileges a user has and adding the same to a new colleague”*.

# Expert Blind Spot at work?

Whaaat, how can such a “one-liner” cause any problems?? Myself being “in the loop” for a decade plus some, was of course
quite surprised. Yeah, assured the companion - took a lot of search and trial-and-error to put together something to extract
all the different object type privileges from the internal catalogues and convert it into something usable. Hugh, just
weird I thought - one could solve it in a few ways - probably he was just “too green” indeed, no point to give a smartass
lecture here, and the conversation went to a new topic...

But on the flight back home (always a nice offline thinking time), the chat somehow circled back to me - have I been
around for too long not to really realise some things anymore? Am I also maybe line-skipping the documentation then, as most
paragraphs actually do look almost the same release-in-release-out? If indeed so, one day I could end up shooting myself
in the foot on some production system…hugh, not a nice prospect. So I decided I’ll try out a little test back home - getting
to the needed output via documentation and googling, through the eyes of a beginner (if possible at all)…

# Cloning a users privileges as a Postgres greenhorn

## First try - Google

Ok let’s start with some standard googling of course - best in incognito mode to (try to) reduce context…

**list all privileges granted to a postgres user**

Gives us…as first a link to an ancient unsupported version (9.0) of Postgres 😀 Which in addition is super generic and
not really helpful at all. And going through the first page of links (assuming one would re-formulate the search if the
first page doesn’t look good) - my mind was blown - indeed, there were no to-the-point links 🤯 Even the most usable
link from Stackoverflow wasn’t directly applicable, meaning easily scriptable. And most links were very affixed only with
table privileges for some reason - although there’s a bunch of other object types in Postgres.

OK, maybe I just picked a bad search phrase (Google probably has a “rewrite engine” similar to Postgres though) - so
let’s try a different one:

**postgres clone privileges of a user to another user**

Ok, already a tad better - the 2nd suggestion at SO, mentioned indeed a way to essentially turn a user into a “role”
(btw, for Postgres they’re the same basically indeed, except ROLEs can’t log in) and hand it out…but this would still
not be exactly perfect - what if our original “role model” gets promoted and suddenly get privs to do anything, while the
“cloned” colleague still only needs to have enough to be able to tick off Jira tickets?


## ChatGPT to the rescue

Ok maybe my googling skills are just sub-par, let’s give our new collective brain a try:

**How to get a listing of all privileges assigned for a user in a PostgreSQL database?**

Nice, we get some solid looking queries - half of which (2 out of 4) even work…but the results are not still actionable
I would say, and only cover table and view privileges and role memberships.

Let’s try another prompt:

**How to clone  all privileges assigned to a PostgreSQL user to a new user?**

Hmm, a bit better - 3 out of 5 queries are even executing! Some even do what they promise they are doing 😀 Still missing
column, schema and function privileges though…

# How I would do it

The 1st thing that came to my mind was of course:

```bash
pg_dump -s "$connstr" | grep 'TO slonik;$' | sed 's/slonik/newuser/g' | psql "$connstr"
```

Which I guess assumes we’ve had some DBA exposure, and have the Postgres package installed - which might be not too common
nowadays - so maybe a more DevOpsy example would look something like:

```bash
docker run --rm postgres pg_dump -s "$connstr" | grep 'TO slonik;$' | sed 's/slonik/newuser/g' | psql "$connstr"
```

By the way - my second idea was to search the non-official Wiki for keywords like “clone privileges”...but that also
didn’t produce anything sadly.

## The most correct solution

Given it was not a one-off task, best would be already to invest some time into creating a proper role hierarchy so that
one could just go with something like:

```sql
GRANT sales_support TO brian;
```

Such examples are fairly nicely also brought out in the official documentation:
[https://www.postgresql.org/docs/17/role-membership.html](https://www.postgresql.org/docs/17/role-membership.html)

# Lessons learnt

To my astonishment - indeed, a very common (and also important, in today's security relevant world) task of listing / cloning all the privileges of a user, is probably not as trivial a task for a beginner as it should be and could probably be improved on. So let’s take it as a friendly reminder that there is always room for improvement, even for the best general purpose database system in the world. And that new blood in the (Postgres) circulation, and communicating with them, is a valuable thing.

Btw, the problem space seems is not so super new actually - from the back of my memory it also surfaced that there’s an
extension called [pg_permissions](https://github.com/cybertec-postgresql/pg_permissions) that tackles similar…so maybe
there’s something to it indeed and maybe some kind of new, auxiliary, task based documentation corner would make
sense - as the current (generally very good) documentation is indeed a more classical one - syntax and feature details
based. Reviving or better curating the unofficial [Postgres Wiki](https://wiki.postgresql.org/) could of course also be
just enough - anything maintained, with good search is probably the key.

An auxiliary learning is of course also that we shouldn’t yet paste ChatGPT generated queries directly into production 🤖

*PS - feel free to [contact](https://kmoppel.github.io/aboutme/) me if need a bit of help with Postgres - I’ve
been working with Postgres daily for the last 10+ years with some of the most Postgres-heavy companies in Europe.*
