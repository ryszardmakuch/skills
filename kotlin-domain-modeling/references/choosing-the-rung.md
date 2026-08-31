# Choosing the rung: the two mistakes, worked through

`SKILL.md`'s ladder — *type it away*, *hand it back*, *fail fast* — is easiest to
see by watching it go wrong in both directions. This file is the two failures and
the intrinsics you reach for once rung 3 is genuinely the answer. The **stage
names** say *where in the domain* an example sits, never which rung it takes;
they are indexed in [worked-examples.md](worked-examples.md).

## Handing back what should have been typed away

The ladder is easiest to see by watching a runtime check disappear. Take a
small kitchen feature on `kitchen`: estimate when a ticket will be ready, based
on how long its dish takes to prepare.

**Start with the problem.** A first pass keeps prep times as raw minutes and
only catches a bad one deep inside the calculation:

```kotlin
private val PREP_MINUTES: Map<DishId, Int> = mapOf(
    burgerId to 12,
    saladId to -5, // typo — meant 5
)

internal fun estimatedReadyTime(dish: DishId, placedAt: Instant): Instant {
    val minutes = PREP_MINUTES.getValue(dish)
    require(minutes > 0) { "Prep time must be positive, got $minutes for $dish" }
    return placedAt.plus(minutes.toLong(), ChronoUnit.MINUTES)
}
```

The typo sits quietly in a `Map<DishId, Int>` until the first salad order of
the day, at which point `estimatedReadyTime` throws — in the middle of live
service, for a customer who did nothing wrong.

**Resist the tempting-looking fix.** It is easy to reach for the
`PartySize`-style validating factory here too — a `PrepDuration.of(minutes):
PrepDuration?` — and force-unwrap it at the one call site with `!!`, on the
theory that "it's always valid anyway, this map is hardcoded":

```kotlin
// The tempting fix — and the `!!` is the tell that it is the wrong one.
private val PREP_TIMES: Map<DishId, PrepDuration> = mapOf(
    burgerId to PrepDuration.of(12)!!,
    saladId to PrepDuration.of(-5)!!, // NullPointerException, no message, no context
)
```

This is worse, not better, and the `!!` is the tell. A reviewer should reject it
— but the useful reading is not "swap `!!` for something safer", it is "the
nullable factory was the wrong tool here".
Follow the three kinds of invalid: `PrepDuration` is *only ever* constructed
from a hardcoded, developer-authored table. No customer, no request payload,
no external caller ever supplies a prep time. Every bad value here is
therefore a caller-contract violation by definition, not a business-reachable
one — so there was never a business-reachable null for a factory to report in
the first place. Take the nullability out and the force-unwrap has nothing
left to bridge.

**Type it away for real**: give `PrepDuration` a plain constructor that
asserts its own invariant, no factory, no nullability anywhere —

```kotlin
data class PrepDuration(val minutes: Int) {
    init { require(minutes > 0) { "Prep time must be positive, got $minutes" } }
}

internal val PREP_TIMES: Map<DishId, PrepDuration> = mapOf(
    burgerId to PrepDuration(12),
    saladId to PrepDuration(5),
)

internal fun estimatedReadyTime(dish: DishId, placedAt: Instant): Instant {
    val duration = PREP_TIMES.getValue(dish)
    return placedAt.plus(duration.minutes.toLong(), ChronoUnit.MINUTES)
}
```

This is a `data class`, not a `value class`, precisely per `SKILL.md`'s
"`value class` or `data class`?": `PREP_TIMES` is a `Map`, so there is no
allocation benefit to protect, and a plain class's constructor has no interop
gap to worry about — `require` here is an unconditional guarantee. Write `PrepDuration(-5)` into that table and it now fails the moment the object is
loaded — in a test, in CI, or at worst on app startup — with the message `require`
gave it, instead of waiting for a specific dish to be ordered during service. `estimatedReadyTime` itself
has nothing left to check: every value in `PREP_TIMES` is already a valid
`PrepDuration` by the time the function runs, so there is no exception and no
result type to write there at all. That is the payoff of pushing the typing-away
as far as it goes — the dilemma does not get resolved, it disappears.

**Handing it back, for comparison**, is the `PlaceOrderResult` sealed type on `ordering`:
"this dish isn't available" or "this table has no active guests" are business
outcomes a caller can act on differently — suggest another dish, ask the host
to seat the party — so they stay in the result type rather than becoming
exceptions or being typed away. There is no way to make "the kitchen sold out
of salad today" structurally unreachable, because it is a fact about the world
that day, not a bug in the code. The difference between this and
`PrepDuration` is exactly the reachability question: one can be caused by the
outside world, the other cannot.

## Failing fast where the caller could have acted

`PrepDuration` was a fail-fast case wrongly dressed in a factory, applied to
something internal-only. It is just as easy to make the opposite mistake:
throwing for something that is actually business-reachable. Take the reservation time on `booking`: like `TableNumber` it
has two conditions and arrives from a customer — but unlike `TableNumber`, its
two failures call for two different answers. A first pass might validate it
inline and let a bad request throw:

```kotlin
// Also wrong: a customer trips either `require` on an ordinary day.
fun bookReservation(rawTime: Instant, clock: Clock, openingHours: OpeningHours) {
    require(rawTime.isAfter(clock.instant())) { "Reservation time must be in the future" }
    require(openingHours.contains(rawTime)) { "Reservation time outside opening hours" }
    // ...
}
```

A customer submitting a booking form can trivially hit either `require` here —
a stale page, a clock skew, picking a time before checking the hours shown
further up the page. None of that is a bug; it is the single most likely way
this function gets called incorrectly, by an ordinary person, on an ordinary
day. Throwing for it means the caller's only option is a generic error page
instead of "please pick a time after 5pm."

**Nothing here can be typed away**, and it is worth saying so explicitly — not
every check can be. Whether a given instant is after "now" is a
fact about the passage of time, not about the shape of the data; no type can
make an already-past `Instant` impossible to construct. So the search
ends quickly, and handing it back is the honest answer:

```kotlin
sealed interface ReservationTimeResult {
    data class Valid(val time: ReservationTime) : ReservationTimeResult
    data object TooSoon : ReservationTimeResult
    data class OutsideOpeningHours(val nextOpenTime: Instant) : ReservationTimeResult
}

@ConsistentCopyVisibility
data class ReservationTime private constructor(val value: Instant) {
    companion object {
        fun of(value: Instant, clock: Clock, openingHours: OpeningHours): ReservationTimeResult {
            if (!value.isAfter(clock.instant())) return ReservationTimeResult.TooSoon
            if (!openingHours.contains(value)) {
                return ReservationTimeResult.OutsideOpeningHours(openingHours.nextOpeningAfter(value))
            }
            return ReservationTimeResult.Valid(ReservationTime(value))
        }
    }
}

sealed interface BookingResult {
    data class Booked(val id: ReservationId) : BookingResult
    data object TooSoon : BookingResult
    data class OutsideOpeningHours(val nextOpenTime: Instant) : BookingResult
    data object NoTableFree : BookingResult
}

fun bookReservation(
    rawTime: Instant,
    partySize: PartySize,
    clock: Clock,
    openingHours: OpeningHours,
    tables: TableBook,
): BookingResult =
    when (val time = ReservationTime.of(rawTime, clock, openingHours)) {
        ReservationTimeResult.TooSoon -> BookingResult.TooSoon
        is ReservationTimeResult.OutsideOpeningHours ->
            BookingResult.OutsideOpeningHours(time.nextOpenTime)
        is ReservationTimeResult.Valid ->
            tables.reserve(time.time, partySize)
                ?.let { BookingResult.Booked(it) }
                ?: BookingResult.NoTableFree
    }
```

**Why `data class` and not `value class` here**, when `TableNumber` and
`PartySize` both got one: an `Instant` is not primitive-shaped, so there is no
allocation to avoid and nothing for `@JvmInline` to buy — and this value
arrives from outside, which is exactly where the interop gap would be the
wrong risk to take. Both of `SKILL.md`'s tests point at `data class`.

**Why `@ConsistentCopyVisibility`**, and which flag closes `copy()` where the
check cannot live in `init`: under "Closing `copy()` when the check cannot live in
`init`" in [worked-examples.md](worked-examples.md).

**Why two sealed types rather than one**, because the wrong version of this is
a real trap. `ReservationTimeResult` is what *one factory* knows;
`BookingResult` is what the *whole use case* can answer with — it carries an
outcome the factory cannot produce (`NoTableFree`) and a success payload the
factory does not have (an id, not a time). That is what earns the second type.
If it had instead come out as a variant-for-variant rename of the first, with
a `when` whose only job was translating each case into an identically-shaped
one, the honest move would be to delete it and return `ReservationTimeResult`
directly: a mapping layer that adds no information is exactly the ceremony the
ladder warns about.

This is deliberately richer than `PartySize.of(): PartySize?` — a bare `null`
would tell the caller *that* the request failed but not *why*, and here the
two reasons genuinely call for different UI responses ("pick a later time"
versus "we open at 5pm, want that instead?"). Notice there is no force-unwrap
anywhere, and no condition is checked twice: `ReservationTime.of` is the *one*
place that knows what makes a time valid, and `bookReservation` pattern-matches
its answer instead of re-deriving it.

The tempting alternative is a nullable `ReservationTime.of` plus the same two
`if` checks re-run inside `bookReservation` to work out *why*, bridged by a
force-unwrap once both pass. The force-unwrap bypass already rules that out on
its own; the duplication is the second, quieter problem.
Someone later adds a third condition to the factory — a public holiday, a
minimum-notice window — updates it, and forgets the copy sitting in
`bookReservation`. If two pieces of code have to agree on a condition, make
them the same piece of code.

One thing this example deliberately does *not* do: aggregate. `TooSoon` and
`OutsideOpeningHours` are mutually exclusive for a single instant, so
returning the first that matches loses nothing. Where a request carries several
*independent* fields — a time and a party size and a table — the boundary
should report every violation at once; that shape is in
[deserialization.md](deserialization.md).

## Which intrinsic, once you have decided to fail fast

`SKILL.md` gives the one-line rule — `require` for an argument, `check` for the
receiver's own state, `error` for the unreachable. These are the consequences,
which is where the four stop being interchangeable:

| Use | When | Throws |
|---|---|---|
| `require` / `requireNotNull` | An **argument** violates the function's contract | `IllegalArgumentException` |
| `check` / `checkNotNull` | The **receiver's own state** is wrong for this call; the arguments are fine | `IllegalStateException` |
| `error(msg)` | Unconditional — a branch that should be unreachable | `IllegalStateException` |

**An `init` block validating constructor parameters is `require`, always** —
parameters are arguments, whatever the class does with them afterwards. This is
the one people get wrong, because the class feels like a receiver; it isn't one
yet, since it is still being constructed.

**Where `error` guards a branch that is dead because a `when` is exhaustive over
a sealed hierarchy, delete the branch instead of writing the `error`.**
Exhaustiveness already proves what the `error` was asserting, and the dead branch
outlives the proof: add a variant later and the compiler stops telling you the
`when` is incomplete, because the `else` you wrote is now handling it.

**`PrepDuration` above is read as typing it away, not as failing fast**, even
though its `init` throws. An asserting constructor does both jobs with one piece
of code: it fails fast for its own caller, and every function that takes a
`PrepDuration` afterwards is relieved of the check entirely. That is what makes
the difference between `!!` and `requireNotNull` more than cosmetic — the second
carries a message, and the first is the same throw with the message deleted.
