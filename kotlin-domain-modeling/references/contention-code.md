# Contention in code: the aggregate, and both strategies

Two callers deciding against the same state. `SKILL.md` carries the decisions —
*read–decide–write*, the three anomalies, and how the strategy shows up in the
result type; this file carries **the Kotlin**. Everything that is a property of one
engine at one version — which states mean contention, what bounds a lock wait, how
stale a replica gets — is in
[contention-operations.md](contention-operations.md).

`cancelling` and `scheduling` are stages of the restaurant flow in
[worked-examples.md](worked-examples.md); *type it away*, *hand it back* and
*fail fast* are the rungs of `SKILL.md`'s ladder, and *detection* and *policy*
are its split of the retry question.

## Contention as a result (`cancelling`)

The canonical transient failure is an optimistic-locking conflict. Take
cancelling a reservation that two operators might click at the same moment, with
a `version` column doing the locking by hand and JDBI doing the talking:

```kotlin
sealed interface CancelReservationResult {
    data object Cancelled : CancelReservationResult
    data object NotFound : CancelReservationResult
    data object AlreadyStarted : CancelReservationResult
    data object Contended : CancelReservationResult
}

/** Takes a `Handle`, so it runs inside whatever transaction the caller opened. */
internal class ReservationStore(private val handle: Handle) {

    fun load(id: ReservationId): ReservationRow? =
        handle.createQuery("SELECT id, status, version FROM reservation WHERE id = :id")
            .bind("id", id.value)
            .map { rs, _ ->
                ReservationRow(
                    id = ReservationId(rs.getString("id")),
                    status = Status.valueOf(rs.getString("status")),
                    version = rs.getInt("version"),
                )
            }
            .findOne()
            .orElse(null)

    /** Rows written: 1 if we won, 0 if the version moved under us. */
    fun cancelAtVersion(id: ReservationId, version: Int): Int =
        handle.createUpdate(
            """
            UPDATE reservation
               SET status = 'CANCELLED', version = version + 1
             WHERE id = :id AND version = :version
            """.trimIndent(),
        )
            .bind("id", id.value)
            .bind("version", version)
            .execute()
}
```

The important thing about that `UPDATE` is what it *doesn't* need: **there is no
exception to catch.** A conditional write guarded on the version column reports
contention as a row count, and a row count is an ordinary return value. That's
typing away a transient failure — the exception-vs-result question doesn't get
answered, it stops being asked.

```kotlin
internal class CancelReservation(private val jdbi: Jdbi) {

    fun handle(id: ReservationId): CancelReservationResult =
        try {
            jdbi.inTransaction<CancelReservationResult, Exception> { handle -> attempt(handle, id) }
        } catch (e: JdbiException) {
            if (e.sqlState() in CONTENTION_STATES) CancelReservationResult.Contended else throw e
        }

    private fun attempt(handle: Handle, id: ReservationId): CancelReservationResult {
        val store = ReservationStore(handle)
        val row = store.load(id) ?: return CancelReservationResult.NotFound
        if (row.status != Status.BOOKED) return CancelReservationResult.AlreadyStarted
        return if (store.cancelAtVersion(id, row.version) == 1) {
            CancelReservationResult.Cancelled
        } else {
            CancelReservationResult.Contended
        }
    }
}
```

**Two sources of `Contended` in one class, and they are not redundant.** The row count
is the write we guarded ourselves; the `catch` is everything the engine raised on its
own, and `CONTENTION_STATES` is *detection* — the set only you can write, in
[contention-operations.md](contention-operations.md). One `catch`, not two: JDBI raises
the same type whether the statement or the commit failed.

Four choices in that shape, and not one of them is incidental:

- **The transaction is opened and committed where you can see it.**
  `jdbi.inTransaction` begins, commits on a normal return and rolls back on a
  throw, all in the function that decides whether a failure is retryable. Nothing
  about that decision is implied by an annotation elsewhere, and there is no
  proxy boundary to reason about.
- **The store takes a `Handle`, not a `Jdbi`.** That is what makes "which
  transaction am I in" unambiguous: it is the caller's, always. A store that
  opens its own connection cannot be composed into a decision.
- **The zero-row branch is unambiguous here, and that's a design choice.** In
  general `0` could mean "no such row" or "version moved", but `load` already
  established the row exists inside this transaction, and `version` increments on
  every write — so the only way to write zero rows is that somebody else wrote
  first. Guarding the `UPDATE` on `version` alone, rather than on `version` plus
  a status predicate, is what keeps it that way; add predicates and you
  reintroduce the "why was it zero?" question and need another `SELECT` to
  answer it.
- **`Contended` doesn't name the database.** Callers branch on "this conflicted,
  try again", never on JDBI, an error code, or a driver class. Changing the store
  becomes a change to this file alone.

### The retry splits in two: *detection* and *policy*

The version guard above needs neither. What needs both is everything the *engine*
raises on its own — a serialization failure, a deadlock victim, a lock that could
not be taken — because those arrive as exceptions rather than as a row count.

- **Detection** answers *was this contention?* It is a property of your engine and
  driver, it is yours to write down, and no library writes it for you. It lives
  in the class that owns the store, which is why no `catch` appears above. The
  states, the helper that digs one out of a wrapped exception, and the measured
  trap in JDBI's own retry runner are in
  [contention-operations.md](contention-operations.md).
- **Policy** answers *how do we retry?* Attempts, backoff, jitter. It comes from a
  retry library — resilience4j is one — where the algorithms already exist and
  have been thought about under load.

Both answers leave the signature alone. `Contended` is in the result type either
way: a caller whose policy has run out of attempts still has to be told, and a
caller with no policy at all lets the HTTP edge answer `409`. **What is not fine
is a signature that promises neither and lets a persistence exception escape into
calling code.**

**Whichever you pick, a retry re-decides — it does not re-write.** Going back to
the read is what lets the next attempt win: re-running the write alone puts the
same stale hypothesis against a version that has already moved, so it fails
identically every time, burns the budget, and reports contention where a re-read
would have found a plain business answer. *Re-decide* is the whole loop —
read, decide, write conditionally — and everything in it runs again on every
attempt. Which is also what makes idempotence a question you have to ask of the
specific process: if this were retried right now, would an outbound call, an
email, or a charge fire twice?

## A rule spanning several objects (`scheduling`)

"One table cannot hold two overlapping reservations" belongs to no single
reservation. Put the rule where the data it needs already is — a table's schedule
for one service date — and it becomes an ordinary check with no query in it:

```kotlin
data class TimeRange(val start: Instant, val end: Instant) {
    init { require(end.isAfter(start)) { "A time range must end after it starts: $start to $end" } }

    fun overlaps(other: TimeRange): Boolean = start.isBefore(other.end) && other.start.isBefore(end)
}

sealed interface BookOutcome {
    data class Booked(val schedule: TableSchedule, val booking: Booking) : BookOutcome
    data class Overlaps(val existing: Booking) : BookOutcome
    data class PartyTooLarge(val capacity: Seats) : BookOutcome
}

data class TableSchedule(
    val tableNumber: TableNumber,
    val capacity: Seats,
    val bookings: List<Booking>,
    val version: Int,
) {
    init {
        require(bookings.distinctBy { it.id }.size == bookings.size) { "Duplicate booking id in a schedule" }
        val overlapping = bookings.indices.any { i ->
            bookings.indices.any { j -> i != j && bookings[i].range.overlaps(bookings[j].range) }
        }
        require(!overlapping) { "A schedule cannot be built with overlapping bookings" }
    }

    fun book(id: ReservationId, range: TimeRange, partySize: PartySize): BookOutcome {
        if (partySize.value > capacity.value) return BookOutcome.PartyTooLarge(capacity)
        bookings.firstOrNull { it.range.overlaps(range) }?.let { return BookOutcome.Overlaps(it) }
        val booking = Booking(id, range, partySize)
        return BookOutcome.Booked(copy(bookings = bookings + booking), booking)
    }
}
```

**The aggregate asserts its own rule in `init`, and that is not the enforcement**
— it is the assertion that nothing got in around `book`. By the time a schedule
is loaded, overlapping bookings would already be committed, so this `require` is
a caller-contract violation: rung 3, deliberately, on a class whose whole job is
rung 1 for its callers.

**`version` on the aggregate is what makes the check mean anything.** Two callers
that both loaded a free schedule both pass `book`, because both decided against
state that was already stale. The write has to be conditional on the version they
read:

```kotlin
// On TableScheduleStore, alongside the plain `load` its optimistic caller uses.
/** Rows written: 1 if we won, 0 if the version moved under us. */
fun appendAtVersion(on: LocalDate, schedule: TableSchedule, booking: Booking): Int {
    val claimed = handle.createUpdate(
        "UPDATE table_schedule SET version = version + 1 " +
            "WHERE table_number = :table AND service_date = :on AND version = :version",
    )
        .bind("table", schedule.tableNumber.value)
        .bind("on", on)
        .bind("version", schedule.version)
        .execute()
    if (claimed == 0) return 0

    insert(on, schedule, booking)
    return 1
}
```

The loser gets `Contended` and re-decides; this time `book` returns `Overlaps`.
Same machinery as `cancelling` above, applied to a rule rather than to a status —
the use case that drives it sits alongside its pessimistic twin below.

**Notice what that `UPDATE` is for.** It writes no data anybody reads — it exists
so the two transactions collide on a row. Two bookings at different times are
disjoint rows, and disjoint rows do not conflict; bumping the schedule's version
is what turns write skew into a plain lost update, which a version guard can see.
**A rule with no row to represent it has nothing to guard on** — that's the
phantom row in the table above, and it holds regardless of which strategy you
picked.

Four things decide whether that aggregate enforces anything at all:

- **Keep the aggregate small enough to lock.** Per table per service date is
  fine. One schedule for the whole restaurant would put every booking in the
  building behind one version.
- **Without the conditional write the check is decorative.** Check-then-write on
  a stale read passes twice and commits twice, and the aggregate can then no
  longer be loaded without violating its own `init`.
- **A database constraint changes job rather than disappearing.** With the rule
  inside the aggregate, an exclusion constraint over a time range is no longer
  the enforcement; it is the assertion that no path writes around the aggregate.
  If it fires, that is a bug, so let it fail fast rather than translating it
  into an outcome. Whether your engine can express that constraint at all is one
  of [contention-operations.md](contention-operations.md)'s questions.
- **Where no lockable unit contains the rule, it is not an invariant.** A guest's
  reservations across restaurants you don't own cannot be made consistent by any
  constructor or any lock. Detect and compensate, and say so in the signature
  rather than shipping a check that reads authoritative.

## Four ways to close the gap, and what each costs

| Approach | The gap between deciding and writing | Costs you |
|---|---|---|
| **Type it away** | none — the fact is about the value, so it cannot go stale | only works for rules a value can carry alone |
| **Pessimistic lock** — an exclusive read | excluded: no second decider exists | a row lock held for the whole decision; waiting, timeouts and deadlocks under load |
| **Optimistic lock** — version guard | detected at the write | a `Contended` variant, and a retry that must re-decide |
| **Constraint or atomic write** — unique, exclusion, upsert | there is no gap: the write *is* the decision | only rules the constraint language can express |

Both middle rows, written out for the same rule, so the difference is the code
rather than the description — and both answer with the same type, which is where
the difference shows up:

```kotlin
sealed interface ReserveTableResult {
    data class Booked(val id: ReservationId) : ReserveTableResult
    data class Overlaps(val existing: ReservationId) : ReserveTableResult
    data class PartyTooLarge(val capacity: Seats) : ReserveTableResult
    data object NoSuchTable : ReserveTableResult
    data object Contended : ReserveTableResult
}
```

The optimistic shape — a plain read, a conditional write, and a `Contended` a
retry policy can act on:

```kotlin
// On ReserveTableOptimistically, which holds only a `Jdbi`.
fun handle(/* ... */): ReserveTableResult =
    jdbi.inTransaction<ReserveTableResult, Exception> { handle ->
            val store = TableScheduleStore(handle)
            // Nothing is held, so this snapshot can go stale under us between
            // deciding and writing. The version we read is what catches that.
            val schedule = store.load(tableNumber, on)
                ?: return@inTransaction ReserveTableResult.NoSuchTable

            when (val outcome = schedule.book(id, range, partySize)) {
                is BookOutcome.PartyTooLarge -> ReserveTableResult.PartyTooLarge(outcome.capacity)
                is BookOutcome.Overlaps -> ReserveTableResult.Overlaps(outcome.existing.id)
                is BookOutcome.Booked ->
                    if (store.appendAtVersion(on, schedule, outcome.booking) == 1) {
                        ReserveTableResult.Booked(id)
                    } else {
                        ReserveTableResult.Contended  // somebody wrote first; re-decide
                    }
            }
        }
```

The pessimistic shape, for the same rule — the read holds the row, so there is
no version to guard on:

```kotlin
// On ReserveTableExclusively — same rule, same aggregate, gap closed by exclusion.
fun handle(/* ... */): ReserveTableResult =
    jdbi.inTransaction<ReserveTableResult, Exception> { handle ->
            val store = TableScheduleStore(handle)
            // Intended to block until any other holder commits: from here to
            // commit the row is ours, so the check cannot go stale in between.
            val schedule = store.loadForUpdate(tableNumber, on)
                ?: return@inTransaction ReserveTableResult.NoSuchTable

            when (val outcome = schedule.book(id, range, partySize)) {
                is BookOutcome.PartyTooLarge -> ReserveTableResult.PartyTooLarge(outcome.capacity)
                is BookOutcome.Overlaps -> ReserveTableResult.Overlaps(outcome.existing.id)
                is BookOutcome.Booked -> {
                    store.append(on, schedule, outcome.booking)  // no version to guard on
                    ReserveTableResult.Booked(id)
                }
            }
        }
```

**Both are wrapped in the same `catch` the cancelling use case has**, for the same
reason — and on the exclusive one that is the point rather than a detail: exclusion took
`Contended` off the happy path, not out of the type, because a bounded lock wait still
ends in `55P03` under load. What the set has to name is in
[contention-operations.md](contention-operations.md).

`loadForUpdate` is the one thing the pessimistic class does which the optimistic
one doesn't, so it is worth showing rather than naming — its plain `load` sibling
is an ordinary `SELECT`, the shape `ReservationStore` shows above:

```kotlin
// Also on TableScheduleStore. The lock is the last clause, taken on the row that
// stands for the rule.
fun loadForUpdate(tableNumber: TableNumber, on: LocalDate): TableSchedule? {
    handle.createQuery(
        "SELECT version FROM table_schedule " +
            "WHERE table_number = :table AND service_date = :on FOR UPDATE",
    )
        .bind("table", tableNumber.value)
        .bind("on", on)
        .mapTo(Int::class.java)
        .findOne()
        .orElse(null) ?: return null
    return load(tableNumber, on)
}
```

**Both strategies point at the same row.** The lock lands on `table_schedule` —
the row `appendAtVersion` bumps the version of, the one that exists only to
represent the rule. Which is why *a rule with no row to represent it* costs you
both middle rows of the four-way table at once, and not just one of them.

**That last clause is where four more decisions land.** A plain exclusive read, a
no-wait variant and a skip-locked variant are three different questions to ask
the database; which of them your engine offers, how it spells each, and what
bounds the wait on the first are *Taking a pessimistic lock is four decisions* in
[contention-operations.md](contention-operations.md).

**The choice shows up in the result type**, which is why it is a modeling
decision and not a tuning knob. Under exclusion the second caller waits and then
gets `Overlaps` — a business answer — so `Contended` leaves the *happy path*
entirely. Under detection it is on every path, along with a retry policy behind
it.

**And the retry is a re-decide**: an attempt that re-ran `appendAtVersion`
against the schedule it already held would report contention for exactly the case
a re-decide calls `Overlaps`.

Past the result type, the two differ in what they cost you at runtime — and each
row here is visible in the code above:

| | Optimistic (version guard) | Pessimistic (an exclusive read) |
|---|---|---|
| Favours | rare contention | a short critical section under heavy contention |
| Retry cost | grows with contention, and each attempt re-decides in full | none — the second caller queues instead of retrying |
| Side-effecting work in between | a retry redoes it | runs once — but the lock is held across it, so a slow call inside becomes everyone else's wait |
| A decision only a human can make | can't be replayed automatically | not available either — needs a compensating flow |

The last row is the one that most often decides it, and it is the same answer on
both sides: **where a person is inside the critical section, neither strategy is
available** and you are choosing between detection and a compensating flow.

## When the read comes from a replica, the rung changes

A retry re-reads from whichever node serves reads. If that is a replica, the
re-read has to outwait replication as well as the competing writer — and that
splits `Contended` into two situations that look identical from inside a retry
and are not the same failure at all:

| | Replication converges | Replication does not converge |
|---|---|---|
| What is happening | lag is a **delay**: a write at T shows up at T + lag, so a re-read after that sees it | lag grows without bound, or replication has stopped |
| Is retry a real caller action? | yes — it just has to wait long enough | **no** — the target recedes, so no budget reaches it |
| What `Contended` means | honest: somebody wrote first, and the retry went too early | a **lie**: it promises the caller an action that does not exist |
| Which rung | transient failure → hand it back | a fault → **fail fast**, log and alert |

**The non-converging case is not a slower version of the first.** It is the case
the ladder sends to *fail fast*: nothing the caller does helps, so a `Contended`
that keeps arriving is a check that reads authoritative and isn't. Ask what your
retry does when the budget is spent — if the answer is "returns `Contended`", ask
what tells anyone that this one was never retryable.

Which of the two you are in, how far behind that node can be at its worst, and
whether your decision even spans two transactions are questions for your own
stack: [contention-operations.md](contention-operations.md).
