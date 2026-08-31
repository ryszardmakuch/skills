# Reviewing Kotlin for domain modeling

**Read `SKILL.md` first.** This file names findings in its vocabulary — *carrier*,
the three kinds of invalid, the ladder's rungs, *read–decide–write*, *bypass*,
*detection* and *policy* — and in [contention-code.md](contention-code.md)'s
*re-decide*, without redefining any of them. A reviewer who has not read them will
recognise the shapes and miss the reasons.

Read the code backwards from its failure cases. **Three passes, in this order, and
you are done when all three have run**: what a checklist sweeps, what only a
reader catches, and what nobody can settle from the diff at all.

## 1. Run the bypass sweep

[bypasses.md](bypasses.md) carries a table of seven paths that reach an object
without running its check, each with a trigger you look for in the diff. Run it
here rather than restating it — **a row that fires and has not been settled is
the finding**, whether or not the invariant happens to hold today.

**Start with the greppable rows, not with the diff**: `!!` and `@JvmInline` are
plain-text searches over the files the change touched, and a `!!` is the modeling
telling you it went wrong. The rest of the table needs reading.

Three of its rows are the ones a reviewer most often waves through: a `data class`
whose check sits in a factory (`copy()` is generated public whatever the
constructor's visibility), a class whose constructor parameters *all* have
defaults (Kotlin synthesises a no-arg constructor and `init` passes against the
defaults), and a method added to a type the change did not create. The first two
show no symptom in the diff at all; the third shows one and reads as ordinary.

## 2. What only a reader catches

Everything below is settled by reading the Kotlin. Flag each.

### The type does not hold its own invariant

- **A `value class` used as a parameter of a function Java can reach**, or one
  where nobody has asked. Java passes the underlying primitive straight in,
  skipping `init`; "don't know" and "yes" have the same answer, `data class`.
- **A `value class` carrying a second property**, or one held mainly inside a
  generic collection or built reflectively. The allocation saving is gone in all
  three while the interop gap stays, so it is paying a cost for nothing.
- **An enum plus nullable fields** where the cases carry *different* data. Which
  combinations are legal then lives nowhere, and every read becomes a `!!`.
- **And the same decision taken too far the other way**: a `sealed` hierarchy of
  `data object`s carrying no data, or the same data in each. An enum wins there —
  `entries`, `valueOf`, a name that persists, `EnumMap`/`EnumSet` — and a sealed
  type buys nothing back, because a `when` over an enum is checked just as
  strictly. Flag both directions or the review teaches only one.
- **`else ->` in a `when` over a sealed outcome type** — most damagingly in an
  adapter, where it converts every future variant into a generic failure. It is
  the `else` that forfeits the check, on a sealed type and on an enum alike.
- **A `require` in `init` that mentions a status flag and another property.** An
  invariant that changes over a lifetime — the states want to be types, each with
  its fields non-null and a transition as the only way to reach the next. Not a
  finding where the states would carry identical fields: then a flag genuinely is
  enough.
- **An invariant living in serializer configuration** — a custom `KSerializer`, a
  `@JsonCreator` pointing at the factory. It works, and it costs more than a DTO:
  easy to miss in review, easy to lose when someone adds a field. A finding only
  where a separate wire type was available and not taken.
- **The same condition checked in a factory and again in `init`, or at a call
  site**, with nothing keeping the copies in agreement. Write the predicate once
  and have both call it.
- **A factory catching its own `require`.** An exception turned into control flow,
  inside the one type that was supposed to make it unnecessary.

### The fact has no carrier

- **A validated DTO passed downstream instead of a domain type.** A validator
  returns a verdict, not a narrower type, so the type after the check is the type
  before it — every caller below re-checks or trusts a convention.
- **A type whose invariant exists only as a framework annotation**, or `@Valid`
  reaching a domain type rather than a wire type. On the wire shape an annotation
  reports violations to a client; it cannot be what a domain type's validity
  rests on.
- **A check after which the value still has the type it had before** — a
  `validate()`, a separate `FooValidator` — standing in for a type's validity.
  What it returns is beside the point: `Unit`, a `Boolean` and a set of
  violations are all verdicts, all discardable, and none of them is something a
  signature further down can demand. Scanning for the `Unit` case alone walks
  past `fun validate(request: Foo): List<String>`, which is the commoner shape.

### The failure is on the wrong rung

- **An exception class named after an outcome** — `*NotFoundException`,
  `*AlreadyExistsException`, `Insufficient*Exception`. A result variant in
  costume.
- **`require` on a value a customer can trivially supply.** Failing fast where
  the caller could have acted, and it is left with a generic error page.
- **A nullable factory whose every call site is internal.** Handing back what
  should have been typed away — an asserting constructor removes the nullability
  and the runtime check together.
- **A sealed result no caller branches on.** The test is countable: how many
  `when` expressions match on it? None means it should have been a nullable.
- **A wrapper around a value nothing downstream leans on** — a `value class` over
  a string that goes to a logger, a count used once as a loop bound. The
  discipline is for values whose validity other code is entitled to assume without
  re-checking; past that it is ceremony, and ceremony is what makes a reviewer
  stop believing the wrappers that do carry something.
- **`Success` plus `Failure(val message: String)`.** An exception with extra
  steps, and worse than one, because the reason is now untyped text nobody can
  match on. **`kotlin.Result` and `runCatching` are the same finding under a
  standard-library name** — one untyped `Throwable`, no exhaustiveness, so a
  caller cannot branch on outcomes it was supposed to act on differently. (Arrow's
  `Either` with a closed sealed type on the left is this discipline in another
  notation, not a finding.)
- **One result type per field rather than per decision.** Where a boundary
  collapses `TableNumberResult`, `PartySizeResult` and `DishIdResult` into one
  violation list anyway, those types exist only to be unwrapped in the function
  that built them. Nullables plus one aggregate say the same thing, and they
  smart-cast.
- **A result type that renames another variant-for-variant.** If a `when` exists
  only to map each case onto an identically-shaped one, delete the outer type. A
  second type earns its place by carrying an outcome the inner one cannot produce,
  or a payload it does not have.
- **A controller returning only the first validation failure** when the client
  could have fixed several at once — and its mirror, **a boundary aggregating
  violations that are mutually exclusive**, which reports "end before start"
  alongside "start is not a date".
- **An `error` guarding a branch that is dead because the `when` is exhaustive.**
  Delete the branch: the dead one outlives the proof, and silences the compiler
  the day a variant is added.
- **The wrong intrinsic once rung 3 is right.** `check` where an **argument** is
  wrong, `require` where the **receiver's own state** is, and — the one most often
  missed — `check` inside an `init`. Constructor parameters are arguments, so an
  `init` validating them is `require`, always: the class feels like a receiver but
  is not one yet.
- **A `require` message interpolating the value it rejected.** A finding wherever the
  value's sensitivity is unsettled — and the Kotlin type does not settle it, since an
  account number is a `Long`.

The internal-only nullable factory and the `require` a customer trips are the two
directions this most often gets wrong — handing back what should have been typed
away, and failing fast where the caller could have acted. Both are worked through
in full in [choosing-the-rung.md](choosing-the-rung.md). What a boundary should
aggregate, and what it should not, is in
[deserialization.md](deserialization.md).

### The vocabulary leaks

- **`HttpStatus`, `ResponseEntity` or `@ResponseStatus` reachable from a use case
  or a domain type.** The adapter depends on the domain, never the reverse.
- **A `JdbiException`, a driver exception or a broker exception in a signature or
  a `catch`** outside the class that owns that dependency. The rule is not about
  persistence: the class that chose the dependency is the only one that can tell
  its business-reachable failures from its transient ones.
- **`Contended` answered with a 500.** A 500 says "we are broken"; a conflict
  says "nothing is broken, try again". Getting it wrong turns ordinary contention
  into an incident on every dashboard.
- **A mapping to a transport that is entirely mechanical** — every variant with
  exactly one obvious status and nothing else to decide. The sealed type was
  probably designed from status codes inward. A domain's outcomes should be
  nameable by someone who does not know which protocol will carry them.
- **An `Unknown` or `Other` variant on a sealed type consumers compile against
  separately.** An `else ->` wearing a different hat: it converts every future
  outcome into a case they cannot act on, permanently, from the first release.

### Contention, as far as the Kotlin shows

- **A fact about the world established at the edge and spent much later.** A time
  validated against a clock, a balance read before an approval step, a quota
  checked before slow work — *read–decide–write* with the interval left open, and
  no second writer needed for it to be wrong by the time it lands. Ask what
  re-establishes the fact next to the write.
- **A rule over rows that do not conflict, guarded as if they did.** Two bookings
  at different times are disjoint rows, so a version or a lock on *them* holds
  nothing: write skew needs a row that represents the predicate. A change that
  adds a version column and no such row has bought the ceremony without the
  protection.
- **A check-then-insert with nothing that can collide.** The third anomaly: the
  row that would change the answer does not exist yet, so there is nothing to
  version and nothing to lock. It takes a constraint, a lock on a parent row, or
  the strongest isolation the engine offers — and which of those is available is
  pass 3's question, not this one's.
- **A rule spanning several objects checked in a use case and then written
  without a conditional write.** Two callers pass the check on the same stale
  read. A raised isolation level offered in its place is not a fix you can accept
  on sight — that one belongs to pass 3.
- **A retry that re-writes rather than re-decides.** Retrying `appendAtVersion`
  against an aggregate loaded before the conflict can never succeed: the version
  has moved and every attempt fails identically.
- **An exception caught for something a conditional write answers with a row
  count.** A write guarded on a version column reports contention as an ordinary
  return value — nothing to catch and nothing to translate. This is the rung the
  change usually skipped.
- **A retry *policy* hand-rolled where *detection* was all that was owed.** A
  `repeat(n)` loop with no backoff and no jitter, written beside the set of states
  that means contention. The set is yours — no library writes it for you; the
  attempts, the backoff and the jitter are a retry library's, and a loop that has
  neither reports contention it caused itself.
- **A conditional write guarded on the version *and* a status predicate**, whose
  zero row count is then read as contention. Zero now means "version moved" or
  "predicate no longer holds" and the code cannot tell which without another
  `SELECT`. Guard on the version alone and the branch stays unambiguous.
- **A store that opens its own connection** instead of taking the caller's
  `Handle`. "Which transaction am I in" stops having an answer, and the store
  can no longer be composed into a decision.
- **An aggregate big enough that everything queues behind one version** — one
  schedule for the whole restaurant rather than one per table per service date.
- **A lock held across an HTTP call, a message round-trip, or a person's
  thinking time.** An unbounded hold. Where a human is inside the critical
  section neither locking strategy is available, and the change is really
  choosing between detection and a compensating flow.
- **A database constraint treated as the enforcement** where the aggregate
  already owns the rule. It is the assertion that nothing writes around the
  aggregate, so let it fail fast rather than translating it into an outcome.

## 3. What the diff alone cannot settle

Everything above you decide by reading the Kotlin. These you cannot. What a
version guard, an exclusive read or an isolation level actually does is a
property of one engine at one version — sometimes of one table's configuration
within it — and none of that is in the diff.

So the finding here is rarely "this is wrong". It is **"nothing says what this
was checked against"**, and a missing answer is the finding you report. Each item
below is the question you ask instead of an assertion you make; the author was
supposed to have resolved it against
[contention-operations.md](contention-operations.md) and written the answer where
you can see it.

- **Concurrency advice applied with the engine nowhere on the record.** Which
  engine, at which version? A version guard, an exclusive read and an isolation
  level mean different things per database, and settings and guarantees change
  defaults between releases of one. A change that assumes an engine without
  naming it is a finding even where the assumption happens to be right — and so
  is one that assumes an isolation level rather than saying which one is in force
  here, after the pool and the framework have had their say.
- **A mechanism relied on without asking whether this table has it.**
  Transactions, row-level locks and MVCC are not universal even within one
  engine, and a read-only replica may reject an exclusive read outright. The same
  question decides whether an exclusion constraint over a time range is
  expressible here at all, which is what the aggregate's fail-fast assertion
  rests on.
- **A contention catch naming fewer states than the code can raise.** Which codes
  does this engine raise for a serialization failure, for a deadlock victim, for
  a lock that could not be taken — and does the driver expose them unwrapped at
  all, or is the set written by hand? A configured lock timeout with no matching
  state in the catch is a timeout escaping the very thing that was supposed to
  turn contention into a result.
- **A library's retry helper trusted for that whole set.** Which failures does it
  actually retry? A helper built around serialization failures often targets
  exactly one code on purpose and lets the rest out as exceptions. Read it at the
  version in the build file. And is what it retries idempotent — it re-decides,
  so would an outbound call, an email or a charge fire twice?
- **An isolation level raised in place of modeling a row.** Is this engine's
  strongest level optimistic or lock-based? If it detects conflicts and aborts,
  the caller is still owed a retry and `Contended` stays in the signature, so
  raising the level changed the mechanism and not the model. A level named for
  repeatable reads may be snapshot isolation, which permits exactly the anomaly
  the change was trying to close.
- **An exclusive read with nothing said about what bounds the wait.** A lock
  timeout, a statement timeout and a transaction timeout may each apply
  independently, with their own defaults, and which of them this version has can
  differ between releases. When one fires, that is contention — so does
  `Contended` come back into the signature exclusion seemed to remove it from?
  And where several rows are locked, is the order canonical on every path, or is
  it planner behaviour nobody has checked?
- **A decision split across two transactions, with nothing saying which node
  serves the read.** A read-only transaction is the one a router is free to send
  to a replica. How stale can that node be at its worst, and was the retry's
  budget sized against that number or against nothing? A budget that fits inside
  the lag spends every attempt on a snapshot predating the write that rejected
  it, and reports contention where nobody competed.
- **A retry that answers `Contended` when nothing will converge.** What does the
  loop do once the budget is spent? If the answer is "returns `Contended`", what
  tells the caller that this one was never retryable? A stalled replica or a lock
  nobody will release is a fault to log and alert on, and a `Contended` there
  promises an action that does not exist.

## Where the codebase already throws

A change that follows a surrounding convention this skill contradicts is **not a
finding**. `SKILL.md` says to follow it, leave the surrounding code as found, and
report the divergence in the change description. **The absent report is the
finding** — an outcome thrown as an exception with nothing anywhere naming the
variant it would have been.
