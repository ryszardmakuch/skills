# Worked examples: the shapes, and the flow they live on

The **stage names** below say *where in the domain* an example sits, never which
rung of `SKILL.md`'s ladder it takes. An example on `seating` may well hand its
failure back. Where the *rung itself* is the question — a failure handed back
that should have been typed away, or thrown where the caller could have acted —
the examples are in [choosing-the-rung.md](choosing-the-rung.md).

## The restaurant flow

Every example in this skill belongs to one flow through a restaurant, in this
order. When adding or changing an example, place it on one of these stages
rather than inventing a new domain.

| Stage | What happens | Type / result | Illustrates | Lives in |
|---|---|---|---|---|
| `booking` | A guest asks for a table at a time | `ReservationTime` → `ReservationTimeResult` | business-reachable, **two** answers → sealed result | `choosing-the-rung.md` |
| `party` | The guest states how many people | `PartySize` → `PartySize?` | business-reachable, **one** answer → nullable | this skill's `SKILL.md` |
| `seating` | The host seats them at a table | `TableNumber` → `TableNumber?` | two conditions, still one answer | this file |
| `ordering` | A waiter places an order | `PlaceOrderResult` | several business outcomes → sealed result | this skill's `SKILL.md` |
| `kitchen` | The kitchen estimates a ready time, then cooks and serves | `PrepDuration`, `TicketSequenceNumber`, `Ticket` | caller-contract values, and an invariant that changes over a lifetime | this file, `choosing-the-rung.md` |
| `cancelling` | Two operators cancel the same reservation at once | `CancelReservationResult` incl. `Contended` | transient failure, and one result mapped at two edges | `contention-code.md`, `adapters.md` |
| `scheduling` | The same table is booked twice for overlapping times | `TableSchedule` → `BookOutcome` | a rule spanning reservations, and the aggregate that owns it | `contention-code.md` |

Every example below assumes a pure-Kotlin codebase. If a type is reachable as
a parameter from Java, `SKILL.md`'s first question in "`value class` or
`data class`?" changes the answer to `data class`.

## The stand-ins

**Every snippet in this skill is an excerpt, and the names below are stand-ins.**
They carry no invariant the examples rely on: they exist so a signature can be
written down. Reading one as a missing file, or inventing behaviour for it, is
reading it wrong — the type under discussion is always the one being declared.

| Stand-in | Stands for |
|---|---|
| `Booking` | one reservation inside a schedule: an id, a `TimeRange`, a `PartySize` |
| `CookId`, `DishId`, `ReservationId`, `TableId`, `TicketId` | identifiers, opaque |
| `FloorPlan` | the tables that exist, answering `contains(number)` |
| `OpeningHours` | when the restaurant is open, answering `contains(instant)` and `nextOpeningAfter(instant)` |
| `PlaceOrderCommand` | a validated order request, already past its boundary |
| `ReservationRow`, `Status`, `TicketStatus` | a row as read, and the status columns behind it |
| `Seats` | a table's capacity |
| `TableScheduleStore` | the store that loads and writes `TableSchedule`, taking the caller's `Handle` |
| `TableBook` | whatever actually holds a table: `reserve(time, partySize)`, answering with an id or `null` |
| `Ack`, `Problem` | a queue acknowledgement and an HTTP error body |

Everything else a snippet names is declared in this skill, in the language, or in
a library the snippet is about.

## Past the Java question: when `value class` stops paying

The gap the hard question exists for — the KEEP citation, and why no `init` can
close it — is bypass 4 in [bypasses.md](bypasses.md). `SKILL.md` gives the hard
question and the default; these are the cases behind that default, each one
removing the allocation saving `value class` exists for while `@JvmInline`'s
interop gap stays.

- **More than one property.** `value class` caps at one, so an amount plus a
  currency, or a range's start plus end, has to be a `data class`.
- **Held mainly inside a generic collection** — `List<PartySize>`,
  `Map<DishId, PrepDuration>` — rather than passed directly as a parameter.
  The value is boxed into the container, so the per-use-site saving is gone.
  Don't take on the interop gap for a benefit you aren't getting. `PrepDuration`
  in [choosing-the-rung.md](choosing-the-rung.md) is exactly this case.
- **Read or constructed reflectively** — a row mapper, a reflection-based
  serializer. `data class` avoids friction there.
- **Only ever handled as a nullable.** A nullable `value class` is boxed too,
  so a type that never travels as a non-null value is paying the gap for
  nothing. A validating factory returning `T?` is not by itself this case: the
  saving returns everywhere the unwrapped value is passed on afterwards.

The common thread: once a type stops being a bare Kotlin parameter,
`value class` buys less than it looks like, while its `init` is the one that
can be skipped.

## Two more shapes of the three kinds

`PartySize` (`party`) is the simplest possible case: one condition, one external
source. The three kinds hold the same way for messier cases — worth seeing
them applied to something business-reachable but multi-condition, and
something caller-contract but not a hardcoded table.

### `seating` — business-reachable, two conditions, still one answer

```kotlin
@JvmInline
value class TableNumber private constructor(val value: Int) {
    init { require(isPositive(value)) { "Table number must be positive, got $value" } }

    companion object {
        private fun isPositive(value: Int) = value > 0

        fun of(value: Int, floorPlan: FloorPlan): TableNumber? =
            if (isPositive(value) && floorPlan.contains(value)) TableNumber(value) else null
    }
}
```

A host typing a table number into the seating screen, or a QR code still stuck
to a table that was removed in a refit, can produce a number that is
non-positive or simply not on the floor plan — still squarely
business-reachable, still a factory, just checking two conditions instead of
one.

**The two conditions do not live in the same place, and that is the point.**
Being positive is a fact about the value alone, so it goes in `init`, where every
construction path runs it. Being on the floor plan needs a collaborator `init`
cannot have, so it can only live in the factory — which is why the constructor is
private here and public on `PartySize`. A check splitting like this is the normal
case, not a special one: put in `init` everything that can go there, and let the
factory hold only the part that genuinely cannot.

Note what two conditions did *not* do: they did not turn this into a sealed
result. **Count the reasons the caller acts on differently, not the reasons the
check can fail.** Here the response is the same either way — "there is no such
table" — so a bare `null` carries everything there is to say. `booking` — the
reservation time, in [choosing-the-rung.md](choosing-the-rung.md) — is the
contrasting case: two conditions again, but two answers the caller has to tell
apart.

### `kitchen` — caller-contract, generated rather than hardcoded

```kotlin
data class TicketSequenceNumber(val value: Int) {
    init { require(value > 0) { "Sequence number must be positive, got $value" } }
}

private val ticketSequence = AtomicInteger(0)

internal fun nextTicketSequenceNumber(): TicketSequenceNumber =
    TicketSequenceNumber(ticketSequence.incrementAndGet())
```

The value here is *computed* by code — an atomic counter — never supplied by
anything outside the process. No customer, no request payload, no external
system can cause `ticketSequence.incrementAndGet()` to return a non-positive
number. If it ever does, that is a bug in this function or in `AtomicInteger`
itself, not a business outcome — so a plain constructor that asserts is the
right call, with no factory and no nullability anywhere. A value read out of a
hardcoded, developer-authored table lands in this same category by a different
route: not computed, but just as firmly out of the outside world's reach.

### `kitchen` — an invariant that changes over a lifetime

A ticket acquires a cook when cooking starts and a served-at when it is served.
The shape people reach for first puts all of it in one class:

```kotlin
// The rule is a relationship between `status` and two nullables, and nothing holds it.
data class LooseTicket(
    val id: TicketId,
    val status: TicketStatus,          // PLACED, COOKING, SERVED
    val cook: CookId? = null,
    val servedAt: Instant? = null,
)

val nonsense = LooseTicket(TicketId("T-1"), TicketStatus.COOKING, cook = null)  // nobody cooking
```

`init { require(status != COOKING || cook != null) }` would reject that one
combination, and the next one, and the next — while every *reader* still holds a
`CookId?` and reaches for `!!` to get at a value the domain guarantees is there.

**Give each state the fields that are meaningful in it, and nothing else:**

```kotlin
sealed interface Ticket {
    val id: TicketId
    val dish: DishId
    val placedAt: Instant
}

data class Placed(
    override val id: TicketId,
    override val dish: DishId,
    override val placedAt: Instant,
) : Ticket {
    fun start(cook: CookId, at: Instant): Cooking = Cooking(id, dish, placedAt, cook, at)
}

data class Cooking(
    override val id: TicketId,
    override val dish: DishId,
    override val placedAt: Instant,
    val cook: CookId,
    val startedAt: Instant,
) : Ticket {
    init { require(!startedAt.isBefore(placedAt)) { "Cooking cannot start before the ticket was placed" } }

    fun serve(at: Instant): Served = Served(id, dish, placedAt, cook, startedAt, at)
}

data class Served(
    override val id: TicketId,
    override val dish: DishId,
    override val placedAt: Instant,
    val cook: CookId,
    val startedAt: Instant,
    val servedAt: Instant,
) : Ticket {
    init { require(!servedAt.isBefore(startedAt)) { "A ticket cannot be served before cooking started" } }
}
```

Three things changed, and only the first is obvious:

- **`Cooking.cook` is not nullable, so no reader force-unwraps it.** The
  forbidden state is not rejected — it cannot be written down.
- **A transition is the only way to reach the next state.** `Placed` has no
  `serve`, so skipping a step is a compile error rather than a runtime check.
- **The ordering rules stay ordinary constructor invariants.** `startedAt`
  before `placedAt` is a two-property `require` in `init` — the *right* kind,
  because both properties are constructor parameters and neither is a flag.

The tell that you need this shape is a `require` naming a status alongside
another property. The tell that you don't is a state whose fields are identical
to its neighbour's; then a flag genuinely is enough.

## `init` and a nullable factory on the same type

`SKILL.md`'s `copy()` bypass says to put the invariant in `init`. That leaves
the factory a question, and the two shapes people reach for first are both
ruled out elsewhere in this skill: catching your own `require` turns an
exception into control flow, and restating the condition in the factory is
"the same condition checked twice, with nothing keeping the two in agreement".

Write the predicate once:

```kotlin
data class ChargingTariff(val minorUnits: Long, val currency: String) {
    init { require(isValid(minorUnits, currency)) { "Invalid tariff: $minorUnits $currency" } }

    companion object {
        private fun isValid(minorUnits: Long, currency: String) =
            minorUnits > 0 && currency.length == 3

        fun of(minorUnits: Long, currency: String): ChargingTariff? =
            if (isValid(minorUnits, currency)) ChargingTariff(minorUnits, currency) else null
    }
}
```

`init` covers every construction path including `copy()`; `of` answers the
business-reachable case with a `null`; the condition exists in one place. The
constructor can stay public here — `init` holds the line regardless of who
calls it.

This applies when the check needs nothing but the constructor parameters.
When it needs collaborators the factory has and `init` cannot — a clock, a
floor plan, a price list — validation has to live in the factory, and the
`copy()` hole is closed with visibility instead. `ReservationTime`, in
[choosing-the-rung.md](choosing-the-rung.md), is that case.

### Closing `copy()` when the check cannot live in `init`

**Why `@ConsistentCopyVisibility`**: the validation cannot live in `init`,
because it needs a `Clock` and the `OpeningHours` that only the factory has.
That is precisely the case the `copy()` bypass warns about — a `private
constructor` with the check in the factory, where a generated public `copy()`
would let anyone holding one valid `ReservationTime` mint an invalid one. The
annotation makes the compiler close that door.

**Let the compiler tell you whether you still have to.** It warns on a
non-public primary constructor exposed through `copy()`, and while that warning
fires the narrowing is opt-in — either `@ConsistentCopyVisibility` on the class,
as that type carries, or `-Xconsistent-data-class-copy-visibility` module-wide. **Prefer the
module flag** where you control the build: one decision instead of one per
class, and no way to forget it on the next type. Treat the warning as the source
of truth rather than a version number — it is still firing, and `copy()` still
generated public, on current Kotlin. Once it stops, the annotation stops being
needed and `@ExposedCopyVisibility` becomes the escape hatch for a class that
deliberately wants a wider `copy()`.

None of that is the first move, though. **The version-independent fix is to put
the invariant in `init`**, which `copy()` then honours with no flag and no
annotation; visibility is what you reach for only when the check needs a
collaborator `init` cannot have, as `ReservationTime`'s does.
