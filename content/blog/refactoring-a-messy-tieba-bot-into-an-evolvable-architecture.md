+++
title = "Refactoring a messy Tieba bot"
date = "2026-03-18"
slug = "refactoring-a-messy-tieba-bot-into-an-evolvable-architecture"
tags = ["notes"]
+++

This bot did not get hard because of one big bug.

It got hard from many small, reasonable choices.

At first the runtime was a simple loop: fetch new threads and replies, filter, judge, maybe like, maybe reply, then save. That lasted longer than it should. Trouble started when new needs blurred what the loop was doing.

The question was no longer “does it run?” It was “can we still say why it did, or did not, do something?”

![Diagram of the old single-pass Tieba bot runtime, where fetching, judging, acting, notifying, and persistence all lived in one loop.](https://img.note.pp.ua/file/blog/articles/tieba-architecture/old-single-pass-runtime.svg)

## The code still runs. The meaning is already off.

The first pressure was meaning, not scale.

Action logic started simple: do not like twice, do not reply twice, record what you did. Later the records were not yes/no. They had a life: planned, done, failed, skipped, dry-run.

One “actions” table could no longer answer:

- already done, or only picked?
- if it failed, can we retry?
- does “skipped” count as done?
- is “planned” current state, or history?

A system can keep running after its meaning goes muddy. After that, safe change gets harder every week.

![Diagram showing the difference between a single overloaded actions table and a clearer split between current state and event history.](https://img.note.pp.ua/file/blog/articles/tieba-architecture/action-state-vs-events.svg)

The fix was not a big framework. It was a split:

- `action_state` for what is true now
- `action_events` for what happened

Small change. Easier to read. Clear that this is not “just a status field”.

## Think like a state machine. You do not need an FSM library.

We needed named moves, not better flags.

Once actions meant `planned`, `leased`, `done`, `failed`, and cooldown, the runtime was already a state machine, even if the code did not say so.

A yes/no model cannot cover:

- short downtime vs real failure
- retry vs dead
- current truth vs history
- who holds the lease vs finished work

![Diagram of the action lifecycle as an FSM-style state transition model, including planned, leased, done, failed, cooldown-rescheduled, and dead.](https://img.note.pp.ua/file/blog/articles/tieba-architecture/action-lifecycle-fsm.svg)

We still did **not** need a heavy FSM library. Named states beat a pile of ifs.

Then a second problem showed up: too much lived in one place.

## Hidden coupling

One round did four jobs:

1. find new content
2. judge what to do
3. run side effects
4. notify and save

That is fine at first. It breaks when those steps fail for different reasons. Fetch can fail. AI judge can fail. Action can fail on cooldown or the network.

Those are different failure types. They should not share one loop forever.

Cooldown made this obvious. When cooldown made discovery feel slow, action time was leaking into find time. That is a design smell.

## Do not rewrite everything first

When upstream is unstable, a full rewrite is tempting. It was the wrong first move.

Better:

- cut the transport edge first
- move the read path earlier
- keep action on the old path for a while
- keep a working baseline while the new edge proves itself

![Diagram of the mixed-boundary transition, where the read path moved earlier while action and interaction paths remained temporarily on the old route.](https://img.note.pp.ua/file/blog/articles/tieba-architecture/mixed-boundary-transition.svg)

Not flashy. It ages well. One real edge, without killing a running system.

Then the product question got clearer too.

## Queue good targets. Do not rank the whole world.

One scan could find more than one good target. The “smart” idea: score all of them, always pick the best. That turns the product into a ranker.

That was not the point.

Better: if more than one target deserves a like or reply, put them in a queue and consume at a sane pace.

Shape:

- ingest / discovery
- `judge_queue`
- judge consumer
- `action_queue`
- action consumer
- state and events

![Diagram of the target queue pipeline, including judge and action queues, consumers, and the split between transaction-first local handoff and idempotency-first external side effects.](https://img.note.pp.ua/file/blog/articles/tieba-architecture/queue-pipeline-and-reliability-boundaries.svg)

SQLite was enough. No Kafka, Redis, or fake broker. We needed a durable local queue: pending, leased, done, dead-letter, retry, cooldown, crash recovery.

A solid local task table did the job.

Queues were not enough. Each edge still needed a reliability rule.

## Transactions inside. Idempotency outside.

Transactions and idempotency are not rivals. They solve different things.

- Local table handoff: **transaction first**
- Outside side effects: **idempotency first**

Examples:

- `judge_queue → action_queue` is one local atomic step
- a real Tieba like/comment must stay safe if the process dies and retries

Local consistency and “safe to run twice” are not the same class of failure. A reliable system usually needs both.

Then the last question was how to add the boxes without breaking production.

## The target design only works if the move is safe

The end shape mattered. The path mattered as much.

A practical rollout:

- **Phase 0:** logs, runtime state, service behavior
- **Phase 1:** add `judge_queue`, keep the old path
- **Phase 2:** add `action_queue`, move side effects off direct judgement
- **Phase 3:** switch defaults only after watching real runs

![Diagram of the phased migration path, showing why architecture changes needed a rollback-safe sequence instead of a big-bang rewrite.](https://img.note.pp.ua/file/blog/articles/tieba-architecture/phased-migration.svg)

That kept the work honest. We did not “upgrade everything”. We moved complexity only when something real pushed.

## What actually changed

The win was not “more advanced”.

The win was: we could explain it again.

Easier to tell apart:

- what the runtime does now vs what we wish it did
- current state vs event history
- discovery time vs action time
- local transactions vs outside idempotency
- a staged move vs a big-bang rewrite

The line I keep:

Architecture is not about making a system look smarter. It is about putting complexity where it belongs.
