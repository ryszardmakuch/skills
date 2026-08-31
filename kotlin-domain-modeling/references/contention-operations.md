# Contention against your own stack: the questions only you can answer

What a version guard, an exclusive read, an isolation level or a lock timeout
actually does is a property of one engine at one version — sometimes of one
table's configuration within it. **None of it is asserted here as fact.** The
shapes contention forces on your model are in
[contention-code.md](contention-code.md); this file is the part of the
work your own stack has to answer, and *detection* and *policy* are its split of
the retry question.

## What "resolve these" asks of you, in all four places below

Four clusters of questions follow: before writing any of it, before trusting a
library's retry helper, before raising an isolation level, and before taking an
exclusive lock. **All four carry the same bar, and each is a closed set. A
cluster is *closed* when you can answer every question in it and have written the
answers where a reviewer sees them** — in the change description, or as a
question to whoever knows if there is someone to ask. Not "consider": answer,
**from your engine's documentation at the version you actually run**, not from
memory and not from this file.

**Code that depends on an open cluster does not get written.** Ask the question
and record it; a guessed answer is the one failure mode here that looks exactly
like advice. What each cluster guards is conditional on it.

## Resolve these five before writing any of it

- **Which engine, and which version?** Version matters as much as engine.
- **Which node serves each read, and how far behind can it be at its worst?**
  Reads and writes need not land on the same node, and what decides that
  usually sits outside the use case — a read-only transaction flag, an
  annotation, a proxy, the pool. If reads go to a replica, ask for the *worst*
  staleness, not the average. See below.

  **The single-transaction shape in [contention-code.md](contention-code.md)
  cannot land you there**, which is the cheapest way to tell whether this applies
  to you: one `inTransaction` reads and writes on one connection, so the read
  goes wherever the write goes. You reach the split by loading the aggregate in
  one transaction — a read-only one a router is free to send elsewhere — and
  writing in a second. If your decision spans two transactions, this question is
  live.
- **Does the mechanism you're about to rely on exist for this table, in this
  configuration?** Transactions, row-level locks and MVCC are not universal
  even within one engine — some accept the exact same syntax while enforcing
  none of it, silently, and a read-only replica may reject an exclusive read
  outright. The same question decides whether an exclusion constraint over a
  time range is expressible here at all, which is what the aggregate's
  fail-fast assertion rests on. What does your engine's documentation say about
  *this* table, on *that* node?
- **Which isolation level is actually in force?** Not the one you assume — the
  one your engine defaults to, after a connection pool or a framework has had
  its say. And which anomalies does that specific level permit on that specific
  engine?
- **How does your driver surface a conflict?** As a typed exception, an error
  code, a status string, a row count? Whatever it is, it is the thing you
  translate at the store, and nowhere above it. That is *detection*, below.

### Sizing a retry against replication lag

[contention-code.md](contention-code.md) has the modeling half of this:
where replication converges, lag is a delay and `Contended` is honest; where it
does not, `Contended` is a lie and the rung is *fail fast*. The number is the
part that lives here.

**In the converging case the retry's timing is a function of the worst-case lag,
not of contention.** A tight loop whose whole budget fits inside the lag spends
every attempt on a snapshot that predates the write that rejected it, then
reports contention where nobody competed. That is a number to set, not a broken
strategy — size the wait against the lag you measured, not the one you hope for.

## Detection: which failures mean contention

A conditional write answers this with a row count and needs nothing from this
section. What needs it is everything the engine raises on its own — a
serialization failure, a deadlock victim, a lock that could not be taken.

**No library writes this set for you.** The states that mean contention are
written down once, next to the store that produces them:

```kotlin
internal val CONTENTION_STATES = setOf("40001", "40P01", "55P03")

internal fun Throwable.sqlState(): String? =
    generateSequence(this) { it.cause }.filterIsInstance<SQLException>().firstOrNull()?.sqlState
```

Those three values are the example's engine, not yours: **which codes does your
engine raise for a serialization failure, for a deadlock victim, for a lock that
could not be taken — and does your driver expose them unwrapped at all?** JDBI
wraps every failed statement in `UnableToExecuteStatementException` and
**translates nothing** — no typed hierarchy, no error categories. That is more
honest than it looks: a library that *did* categorise would have to guess, and
any code it left out would look like a bug in your code rather than a gap in its
table.

And once you have the set: it answers *whether it was contention*, not *how to
retry it*. Does each state in it want the same treatment — the same bound, the
same backoff — or does grouping them for detection hide a difference that matters
the moment you decide what happens next?

### A library's retry helper covers less than its name suggests

**Which failures does it actually retry?** Resolve it to the bar above, from the
source or the documentation at the version in your build file, before trusting it
with `Contended`. One measured answer. Don't assume its set matches the one you wrote for
detection above.

**And is what it retries idempotent?** It re-decides, which is the shape you
want — and which is exactly why everything inside the block runs again. Ask it of
the specific process you're modeling: if this were retried right now, would an
outbound call, an email, or a charge fire twice?

## Policy: how many attempts, and how far apart

**A retry policy comes from a retry library** — resilience4j is one — where
bounded attempts, exponential backoff and jitter already exist and have been
worked out under load. Its budget is a number you set against the lag and the
contention you measured, and a caller that exhausts it still sees `Contended`.

## Raising the isolation level is not the third option it looks like

Reaching for the strongest isolation level instead of modeling a row for the
predicate looks like a way to skip the work. Resolve these four, to the bar
above, before believing it:

- **Is your engine's strongest level optimistic or lock-based?** Some implement
  it by detecting conflicts and aborting a transaction that must then
  re-decide — the same bargain a version guard makes, moved into the engine, and
  `Contended` stays in your signature. Others block instead. The answer decides
  whether you still owe the caller a retry, so it is a modeling question, not a
  configuration one.
- **What can it raise beyond the conflict you expect?** A stronger level does
  not always prevent errors that could not occur in a true serial execution — a
  constraint violation can still surface after a check that looked safe. Does
  your detection handle more than the one exception shape you planned for?
- **Are you stacking an explicit lock on top of it?** If the level already
  provides the protection, an added lock is cost rather than safety. Does your
  engine's documentation say the two compose, or that one makes the other
  redundant?
- **What does it cost on the queries you are serializing?** Conflict detection
  has to track what each transaction read, and how precisely it can do that
  often depends on your indexes. Is a flood of retries genuine contention, or
  the shape of your query?

## Taking a pessimistic lock is four decisions, not one

An exclusive read reads like one keyword. Resolve these four, to the bar above,
before writing it — each is a way it goes wrong in production rather than in a
test:

- **What actually bounds the wait, and what does hitting that bound mean?** An
  unbounded lock wait can park every worker in a bounded connection pool and
  turn one hot row into a whole-service outage rather than one slow endpoint.
  But *which* setting bounds it is not one answer: a lock-specific timeout, a
  statement timeout and a transaction timeout may each apply, independently,
  with their own defaults, and which of them your version even has can differ
  between releases. Deadlock detection may not bound a plain wait at all — it
  may only fire on a real cycle. So: on your engine and version, which of these
  is in force for this statement, at what value, after your pool and framework
  have set theirs? And once one does fire — **that is contention**, so does
  `Contended` come back into the signature exclusion seemed to remove it from?
- **Are you acquiring locks in a deterministic order?** A deadlock needs a
  cycle; a canonical order — sort by primary key — removes it instead of
  detecting it. If a transaction locks several rows, is the order the same on
  every path that reaches it? And if you're relying on one statement's `ORDER
  BY` to guarantee the order locks are taken in, does your engine document that
  as a guarantee, or is it planner behaviour you cannot lean on?
- **Do you want this row, or any row nobody is holding?** Engines offer a
  variant that fails immediately rather than waiting — which puts you back on
  detection, with a `Contended` variant, having paid for a lock — and one that
  skips locked rows entirely, changing the question to "give me an unheld row".
  The second is right for a job queue and wrong wherever a specific row (this
  table's schedule) is the only acceptable answer. Which question does your
  case actually need answered?
- **Is anything inside the locked section outside your control?** A lock held
  over an HTTP call, a message round-trip, or a person's thinking time is a
  lock held for an unbounded time. If a human is part of the decision, is
  exclusion even available here, or are you actually choosing between detection
  and a compensating flow?
