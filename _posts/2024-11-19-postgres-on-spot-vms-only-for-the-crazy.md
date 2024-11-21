---
layout: post
title: Postgres on Spot VMs - only for the crazy?
cover-img: /assets/img/elephant-sitting-on-a-tree.jpeg
tags: [postgres, spot, aws, self-managed, ansible, cost-saving]
---

Postgres is already great, surely - even "too popular" one could complain with a twist...as this broadcast "was"
actually supposed to be my Lightning Talk at last month’s pgConf.eu in Athens 🙂 But indeed,  seems Postgres has in an
awesome way gotten so big that it’s not like that anymore that you show up and get a LT slot - there was a crazy 3x
overbooking of applicants, thus a draw had to be made, and well - wasn't my lucky day…so need to hit the keyboard a bit
I guess instead 🙂

# The Lighting Talk that wasn't

If have have a minute or two to spare, I'd recommend to check out the full slides at:
[https://kmoppel.github.io/assets/media/postgres_on_spot_vms_lightining_talk.pdf](https://kmoppel.github.io/assets/media/postgres_on_spot_vms_lightining_talk.pdf)

Anyways, the pun of the talk was supposed to be something like that:

* Postgres is already great - everyone in tech knows that based on all kinds of surveys!
* But wait - what if one could make it even 5x better! Crazy talk???
* Am I talking about:
  - Async IO?
  - Or perhaps vectorized instructions?
  - Or a new columnar engine?
* Nope: The Elephant in the room - managed database costs…
* The answer, for some use case only of course: utilising Spot instances! That are up to 10x cheaper than on-demand
  instances…but nevertheless can run months uninterrupted if chosen wisely.
* And to make things even simpler - after using Spot successfully for about 2 years for personal purposes and also for
  clients, I took the time to put together a hopefully more widely useful utility called **[pg-spot-operator](https://github.com/pg-spot-ops/pg-spot-operator)** - which
  make using Spot + Postgres really much more fun 🥳
* And my general take - Spot doesn't only have to mean "for fun & play" - one could also actually use it for more serious,
  (carefully selected) workloads and save a ton of money in the process! And maybe even make the planet a bit more greener,
  who knows…
* Also - a call for some VC dollars, euros, bitcoin - to try to build it into a real product 



# Main idea of the pg-spot-operator

In short pg-spot-operator does the following:

1. Translates human readable hardware requirements to matching (non-human-readable) instance types
1. Launches the Spot VM
1. Runs Ansible to set up Postgres + any extensions / other auxiliary packages
1. Keeps checking that the VM is there
1. If evicted, or some critical user input changes, re-run everything

# Kubernetes?

And yes, in case you wonder - why not to "just" use K8s + Spot node pools? Because in short, it’s not same-same - K8s
needs quite a bit of upfront setup, choosing some instance types, keeping them updated according to price changes, to
name a few. So I’d say for a majority of DB-heavy purposes, ad-hoc Spot is preferred - given of course someone / something
takes care of the operational side, which is the whole point of the project. Will also further elaborate on some future
posting on the pros and cons, compared to K8s or ECS.

# Call for feedback

Github link here once more: [github.com/pg-spot-ops/pg-spot-operator](https://github.com/pg-spot-ops/pg-spot-operator)

And I'd love to [hear](mailto:kaarel.moppel@gmail.com) any kind of feedback or thoughts around such a project, especially
if you have been also running something similar (Spot + Postgres or any relational DB really) on some other clouds besides
AWS - did it pan out?

Or why not just give the repo a star? ⭐ 🙏

Project status I'd say: *Working Beta* - the "API" can change.


*PS - feel free to [contact](https://kmoppel.github.io/aboutme/) me if need a bit of help with Postgres - I’ve
been working with Postgres daily for the last 10+ years with some of the most Postgres-heavy companies in Europe.*
