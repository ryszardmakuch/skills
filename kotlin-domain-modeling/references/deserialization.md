# Deserialization: keeping the invariant at the way in

A value arriving from outside. `SKILL.md` carries the rules; this file carries
the code. `party` and `seating` are stages of the restaurant flow in
[worked-examples.md](worked-examples.md).

A `private constructor` plus a validating factory protects nothing if a
deserializer rebuilds the object some other way — **no deserializer calls your
factory.** `init` is the stronger guard, because it sits on every construction
path the language itself provides.

**Whether your deserializer uses one of those paths is a question about your
library, not about Kotlin.** Resolve these three before leaning on `init`, and
resolve them with a test on the version in your build file rather than from
memory:

- **Does it construct through a constructor at all**, or allocate and set fields
  reflectively? Only the first runs `init`.
- **What does it do with a class it cannot construct** — fail loudly, or fill in
  defaults and carry on?
- **Has that answer changed between your version and the one whoever told you
  was running?** This is behaviour libraries change.

For what it is worth as a sample of two: kotlinx.serialization 1.11.0 runs
`init`, including through the synthetic constructor it generates for defaulted
properties, and Jackson 2.22.1 without `jackson-module-kotlin` fails loudly
rather than quietly filling in fields. Those are answers for those versions.
Get yours the same way they were got — from a test.

**One shape escapes all three questions, and it is the trap.** Where every
constructor parameter has a default, *Kotlin* — not your library — synthesises a
no-arg constructor that lets `init` pass against the defaults while the payload
lands in the fields afterwards. So no answer to the three above closes it, and no
better deserializer does either. The mechanism is bypass 1 in
[bypasses.md](bypasses.md).

Make the boundary explicit and every one of those questions stops applying:

```kotlin
// Wire shape: primitives only, no invariants, nothing for a deserializer to skip.
@Serializable
internal data class BookTableRequest(
    val tableNumber: Int,
    val partySize: Int,
)

// Domain shape: constructing one of these *is* the validation.
data class BookTableCommand(
    val tableNumber: TableNumber,
    val partySize: PartySize,
)

sealed interface Violation {
    data object UnknownTable : Violation
    data object PartySizeTooSmall : Violation
}

sealed interface RequestParseResult {
    data class Parsed(val command: BookTableCommand) : RequestParseResult

    data class Rejected(val violations: List<Violation>) : RequestParseResult {
        init { require(violations.isNotEmpty()) { "Rejected needs at least one violation" } }
    }
}

internal fun BookTableRequest.toCommand(floorPlan: FloorPlan): RequestParseResult {
    val table = TableNumber.of(tableNumber, floorPlan)
    val size = PartySize.of(partySize)

    if (table == null || size == null) {
        return RequestParseResult.Rejected(
            buildList {
                if (table == null) add(Violation.UnknownTable)
                if (size == null) add(Violation.PartySizeTooSmall)
            },
        )
    }
    // Both smart-cast to non-null past that guard — no `!!` anywhere.
    return RequestParseResult.Parsed(BookTableCommand(table, size))
}
```

The deserializer only ever sees `Int`s, so there is no constructor for it to
bypass and no configuration that can silently change the answer. The invariant
lives in exactly one place — the factory and the `init` sharing its predicate —
and the boundary that applies it is a function you can read, test, and point at
in review.

Four things worth noticing:

- **It returns a result rather than throwing.** Bad input at an HTTP or queue
  boundary is the most business-reachable thing there is, so the same rule
  applies here as everywhere else.
- **It reports every violation, not the first.** A request with a bad table
  *and* a bad party size that comes back naming one takes two round trips to
  fix, then a third if the second fix reveals a third problem. Aggregate at a
  boundary a client will act on; inside the domain, first-failure is fine,
  because there is nobody left to show a list to.
- **The violations name fields, not values.** `PartySizeTooSmall` says which field
  and which constraint, so the list travels to a client and to a log without
  carrying the payload it rejected.
- **The single-guard shape is what keeps `!!` out.** Checking `table == null
  || size == null` once, and building the list inside that branch, lets Kotlin
  smart-cast both values afterwards. Writing the same thing as two separate
  `?: run { ... }` blocks is what pushes people into a force-unwrap at the
  end. And `Rejected`'s own `init { require(...) }` is this skill applied to
  itself: an empty violation list is a caller-contract bug, since only this
  file constructs one.

Aggregate only what is **independent**. Where one field's validity depends on
another's — a date range whose end must follow its start — parse the parts,
aggregate those, and check the relationship only once both parts exist. Don't
report "end before start" alongside "start is not a date".

## Where a validation framework fits, and where it does not

The function above already produces the violation list, so an annotation that
restates a domain predicate — `@field:Min(1)` beside `PartySize`'s `value > 0` —
is one rule with two homes and nothing keeping them in step. The asymmetry is
what makes that worth avoiding rather than merely untidy: an annotation
**stricter** than the domain rejects at the boundary what the domain would have
accepted, and nobody finds out; a laxer one is dead code, because the factory
catches it anyway.

**An annotation on the wire type may carry what the domain type does not say** —
that a field is present at all, a length cap that bounds the work before parsing.
Those are facts about the protocol, and no domain type has an opinion about them.

**The line to hold is that `@Valid` never reaches `BookTableCommand`.** On the
wire type an annotation reports violations to a client; on a domain type it would
be what that type's validity rests on, which is exactly the arrangement
`SKILL.md` says a type cannot be built out of.

Wherever a `Validator` is used at all, **build it once and reuse it.** A
`ValidatorFactory` is application-scoped and `AutoCloseable`, and the `Validator`
it hands out is thread-safe;
`Validation.buildDefaultValidatorFactory()` is convenient enough to invite
building one per request, which is the one shape to keep out of a boundary.

The alternatives — a custom `KSerializer`, or `@JsonCreator` pointing at the
factory — do work, and are worth it when you can't introduce a separate wire
type. They cost more: the validation moves into serializer configuration,
where it's easy to miss in review and easy to lose when someone adds a field.
Prefer the DTO when you have the choice.
