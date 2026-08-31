---
name: kotlin-domain-modeling
description: Model Kotlin domain types so an invalid value cannot be constructed, and an anticipated failure comes back as data rather than a thrown exception. Use when putting an invariant on a value class or data class, or closing the copy() hole; choosing value class against data class; choosing require/check/error against a returned nullable or sealed result; modeling an operation's outcomes; modeling a lifecycle whose invariant changes between states; placing a rule that spans several objects; a value crossing a boundary - deserialization, HTTP, a queue; two callers deciding against the same state - optimistic or pessimistic locking, contention, retries; or reviewing Kotlin for any of these.
license: MIT
compatibility: Kotlin projects; the skill needs no tooling. The contention references assert no guarantee for your engine
metadata:
  version: "0.9.0"
  authoring: human-coauthored based on human experience, human-reviewed, drafted and iterated with AI assistance
  disclaimer: Educational content. Read it in full and verify it was applied - an agent may skip a skill entirely. You are responsible for evaluating its advice and the results; no warranty. Full terms in NOTICE.md.
---

# Kotlin Domain Modeling

## The fact and its carrier

A check produces a **fact**: *this party size is positive*, *this table is free at
eight*. The work is choosing **what carries the fact, because the carrier decides how
long it stays true.**

- **The value.** The fact travels with it, on every path. Nothing downstream re-checks,
  because nothing can hold the value without it.
- **A result.** The fact can go out of date, but visibly: a caller who decided against
  state that has moved is told to look again.
- **Nothing.** A `validate()`, a `FooValidator` beside the type. Its **verdict** can be
  as rich as a list of violations and it changes nothing: the value comes back in the
  type it arrived in, so nothing downstream can rest on it.

**Choosing a carrier is choosing a lifetime, and the lifetime sorts the failure:**

| Kind | The fact | What it is |
|---|---|---|
| **Caller-contract violation** | established inside your boundary, so false only if your code is wrong | a bug, not an outcome |
| **Business-reachable** | never established, because the value came from outside | normal under correct operation |
| **Transient failure** | *was* true, and expired before you spent it | nothing broke; retrying is a real action |

**A fact nobody leans on needs no carrier.** This is for values whose validity other code
is entitled to assume without re-checking; a message bound for a logger has nothing
leaning on it, and past that line the discipline turns into ceremony.

`PartySize` holds a fact about an `Int` that nothing can make false; `ReservationTime`
holds one about the world at the instant it was built. Both are correct. **What differs
is what the fact was about** — and that difference is the third kind: contention, which
is therefore not a separate subject from modeling.

## The decision order

Sort the failure, then take the first rung that applies.

1. **Type it away.** Restructure the type or the operation until the bad value cannot be
   written down and no check is left to run.
2. **Hand it back.** The caller has a genuinely different action, so return the failure
   as data — a nullable for one answer, a sealed result for several. **Count the answers
   the caller acts on differently, not the ways the check can fail.** That count settles
   every later question here about which shape to return.
3. **Fail fast.** Nobody up the chain can do anything but abort, so throw: `require` for
   an **argument**, `check` for the **receiver's own state**, `error` for the
   unreachable. An `init` validating constructor parameters is `require` **always** —
   parameters are arguments, whatever the class does with them next.

**Rung 3 is taken too often and rung 1 too rarely** — the two mistakes are failing fast
where the caller could have acted, and wrapping a factory around something no outside
caller reaches. Both in [choosing-the-rung.md](references/choosing-the-rung.md).

**The ladder decides one check at one site.** Typing a value away deletes the check
downstream and leaves one at the edge, which is its own decision, usually to hand it
back. An exceptions-versus-results argument is usually one not yet pushed to the edge.

**You are done when every value your change takes from outside, and every way it can
fail, sits on a rung you can name.** A new type: one place enforces the invariant. A
method added to an existing one: the same question about its parameters, **plus whether
that type's invariant still holds on every path the change added** — a factory, a
mutator, a `copy` site. Adding a method is the ordinary way an invariant nobody touched
stops holding. **And no bypass is open**: a table to run, not a thing to bear in mind —
[bypasses.md](references/bypasses.md).

## *read–decide–write*

**The gap does not need a database.** Validate a reservation time at 18:59:59 for
19:00:00, take two seconds to write, and the write is invalid: the check passed, the fact
expired, nothing raced you. Every rule about state you do not own has that shape — you
**read**, you **decide**, you **write** — and the interval is the problem. Typing it away
is the case where the interval is zero.

Add a second writer to that interval and you have contention. Only the first anomaly is
the plain case a version or a lock covers:

| Anomaly | The shape | What it takes |
|---|---|---|
| **Lost update** | read-modify-write on **one row** | a version on that row, or a lock on it |
| **Write skew** | check-then-act, then write **disjoint rows** | a row that *represents the predicate* — disjoint rows never conflict, so neither strategy has anything to hold |
| **A phantom** | check-then-act, then **insert** a row that changes the answer | a constraint, a lock on a parent row, or the strongest isolation your engine offers |

**A rule spanning several objects tells you where the aggregate goes.** "One table cannot
hold two overlapping reservations" belongs to no single reservation, so the consistency
boundary is in the wrong place. Load, decide and write the unit that owns the rule: a
table's schedule for one service date, inside which "does this overlap?" is an ordinary
question about a list you already hold. That unit is also **the one row two writers
collide on**, which demotes write skew to a plain lost update. **A rule with no row to
represent it has nothing to version and nothing to lock**, so it costs you both
strategies at once.

**The strategy is visible in the result type**, which is what makes it a modeling
decision. Under **detection** — a plain read, a write conditional on the version —
`Contended` is on every path. Under **exclusion** — an exclusive read — the second caller
waits and answers in business vocabulary, so `Contended` leaves the happy path *until you
bound the wait*: a bounded wait that expires is contention, back by another door. A higher
isolation level is not a third option — an engine's strongest may close write skew
*optimistically*, leaving you the retry you were avoiding. Both shapes for the same rule:
[contention-code.md](references/contention-code.md).

**Type it away here too**, the step usually skipped: a write guarded on a version column
reports contention as a **row count**, nothing to catch or translate.

**A retry *re-decides*; it does not re-write.** Re-running the write alone puts a stale
decision against a version that already moved, failing identically until the budget is
gone, where going back to the read answers on the next attempt. Everything in the block
runs again — so ask whether a retry would send the email twice. Split it: **detection** —
*was this contention?* — is your engine's and your driver's, and no library writes it for
you; **policy** — attempts, backoff, jitter — comes from a retry library. `Contended`
stays in the signature either way.

**Apply the ladder's test rather than assuming its answer.** Retrying stops being a real
action when what you wait on will not resolve by itself — stalled replication, a lock
nobody releases. `Contended` then promises an action that does not exist: *fail fast*.

**None of this is safe from memory.** What an isolation level, a row lock or a timeout
guarantees belongs to one engine at one version, as does which node serves your reads:
[contention-operations.md](references/contention-operations.md).

## Modeling the type

**Reach for `sealed class`/`sealed interface` where the cases carry *different data*.**
An enum constant is a singleton with one shared set of properties, so a case needing its
own data hangs it on a nullable the others leave null, and every read becomes a `!!`.

**Exhaustiveness is not the reason.** A `when` over an enum is checked exactly as
strictly, statement or expression, and what forfeits the check is an `else`, on either.
So **an enum still wins** for a fixed set of cases with no data or the same data:
`entries`, `valueOf`, a name to persist, `EnumMap`/`EnumSet`.

**An invariant that changes across a lifetime needs more than one type.** "A ticket has
a cook once it is cooking" cannot be stated by one constructor. **Give each state its own
type carrying only the fields meaningful in it, all non-null, and make each transition a
function to the next state** — `Placed.start(cook, at): Cooking`, under a sealed root for
callers holding "a ticket". Skipping a step becomes a missing method rather than a
runtime check. The tell: an `init` whose `require` names a status alongside another
property. Under `kitchen` in [worked-examples.md](references/worked-examples.md).

**`value class` or `data class`?** Settle it before writing the type: both take
`init { require(...) }`, so this is representation and interop, never validation.

> **Is the type a parameter of any function Java code can reach?**
> Yes → `data class`. No → `value class` is fine. Don't know → `data class`.

Java can pass the underlying primitive straight in, skipping the constructor and its
`init` — the one bypass with no fix inside the type. Past that, when in doubt `data
class`: `value class` stops paying once the value takes a second property, lives in a
generic collection, or is built reflectively
([worked-examples.md](references/worked-examples.md)).

## Enforcing the invariant

**Untrusted says where a value came from, not what shape it is.** Nothing becomes
trustworthy by being parsed, bound by a framework, or sent on an earlier trip through your
own system — so **construction is the last thing that happens at a boundary, not the
first**, and the wire shape decides more than the domain type's invariant.

**Two questions, and running them together is the usual mistake.**

**Where does the check live?** In `init` — it runs on every path the language provides:
`copy()`, a deserializer, a row mapper, a Java caller, a test fixture, where a factory
runs only on the path you wrote. The exception is a check needing a collaborator `init`
cannot have, a clock or a floor plan; then it lives in the factory and `copy()` is closed
with visibility instead. **A private constructor does not close it** — `copy()` is
generated public regardless — which is why reaching for a factory when `init` would do
opens the bypass.

**What does the caller get?** Only a factory can *return*, so this is the question a
factory exists to answer: it does not hold the invariant, it picks the failure mode.
Count the answers, not the failures:

| Answers the caller acts on differently | The caller gets | Example |
|---|---|---|
| **none** | **no factory at all** | a caller-contract bug has nothing to report |
| **one** | a factory returning `T?` | `TableNumber` — *two* rejection conditions, one answer |
| **several** | a factory returning a small sealed result | `ReservationTime` — `TooSoon` and `OutsideOpeningHours` differ to a client |

Write the predicate once and let both paths call it:

```kotlin
@JvmInline
value class PartySize(val value: Int) {
    init { require(isValid(value)) { "Party size must be positive, got $value" } }

    companion object {
        private fun isValid(value: Int) = value > 0

        fun of(value: Int): PartySize? = if (isValid(value)) PartySize(value) else null
    }
}
```

**The two paths never meet.** `of` decides *before* constructing, so an invalid value
returns `null` without the constructor running — the factory is not catching `init`'s
`require`. The constructor stays public: `init` holds the line whoever calls it.

**A message names the field and the constraint.** That is what a reader can act on, and
it is what `Violation.PartySizeTooSmall` in
[deserialization.md](references/deserialization.md) already carries. `got $value` adds
the rejected value to a log, and whether that is safe is a question about the value's
sensitivity, never about its Kotlin type — an account number is a `Long`. Interpolate a
value whose whole range you would publish.

**A nullable is not a weaker sealed result**, and the compiler settles half of it: at a
boundary aggregating several fields a nullable smart-casts past the guard while a sealed
result per field does not. One result type per *field* is a finding in
[review.md](references/review.md).

**A validator returns a verdict; a parser returns a proof.** Holding a validated value
tells you nothing about who validated it. A parser hands back a narrower type, so the
fact travels with the value. Annotations belong on the wire shape:
[deserialization.md](references/deserialization.md).

## Outcomes as sealed types

```kotlin
sealed interface PlaceOrderResult {
    data class Placed(val ticketId: TicketId) : PlaceOrderResult
    data class DishUnavailable(val dishId: DishId) : PlaceOrderResult
    data class TableHasNoActiveGuests(val tableId: TableId) : PlaceOrderResult
}
```

The use case returns that — `place(command: PlaceOrderCommand): PlaceOrderResult` — and
callers answer with an exhaustive `when` the compiler keeps exhaustive as outcomes are
added. A `DishUnavailableException` carries the same information with that help removed.
`TableHasNoActiveGuests` is why both tools coexist: `TableNumber`'s invariant says the
table *exists*, this variant says it is *occupied*, and neither types away into the other.

**Name variants in the domain's vocabulary, never the transport's** — no
`@ResponseStatus` on a variant, no `HttpStatus` in a use case signature. The same "not
found" is a 404 over HTTP and a successful ack on a queue, so no status code is *the*
translation of anything. Translating at an edge, and the same rule for any dependency you
own: [adapters.md](references/adapters.md); every violation rather than the first at the
way in: [deserialization.md](references/deserialization.md).

## Where the surrounding code already throws

Where the local convention contradicts this skill, **follow the convention** — half
sealed results and half exceptions is worse than either done consistently — then **report
the divergence**, naming the outcome thrown and the variant it would have been. **Leave
the surrounding code as you found it.** A **new** module starts here instead.

## References

| File | Load when |
|---|---|
| [review.md](references/review.md) | Reviewing rather than writing: three passes, the last answerable only outside the diff |
| [bypasses.md](references/bypasses.md) | Before calling a change done: paths that reach an object without running its check |
| [choosing-the-rung.md](references/choosing-the-rung.md) | The rung is not obvious: both mistakes in code, and which intrinsic on rung 3 |
| [worked-examples.md](references/worked-examples.md) | The shapes in full: the restaurant flow and its stand-ins, states as types, `init` with a factory, `value class` |
| [deserialization.md](references/deserialization.md) | A value arrives from outside: wire type versus domain type, and what to ask your deserializer |
| [adapters.md](references/adapters.md) | A result leaves the domain for HTTP, a queue, or a published artifact |
| [contention-code.md](references/contention-code.md) | Two callers decide against the same state: the aggregate, and both strategies side by side |
| [contention-operations.md](references/contention-operations.md) | Before writing that against a real database: which engine, which node, what bounds a lock wait |
