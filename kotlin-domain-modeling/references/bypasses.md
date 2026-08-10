# Bypasses: when an enforced invariant silently isn't

A **bypass** is a path that reaches your object without running the check you
wrote. The check is there, the review passed, and the invariant still doesn't
hold — because the language, a library, a build config, or your own modeling
routed around it.

## The sweep

**Run this table against your change before calling it done.** Every trigger is
something you can look for in the code you are about to ship, so this is a
finished check rather than a thing to bear in mind. A row that fires and has not
been settled is a row that is not done.

| Bypass | Fires when your change contains | Settled in |
|---|---|---|
| **Deserialization** | a type a deserializer, row mapper or framework binder constructs | below |
| **`copy()`** | a `data class` whose check lives in a factory rather than in `init` | below |
| **The force-unwrap** | a `!!` | below |
| **Java interop** | `@JvmInline` on a type any Java code can pass a value to | below |
| **The new path** | a factory, a mutator or a `copy` site added to a type your change did not create | below |
| **The `else` branch** | an `else ->` in a `when` over a sealed outcome type | [adapters.md](adapters.md) |
| **The unknown variant** | an `Unknown` or `Other` variant on a sealed type consumers compile against separately | [adapters.md](adapters.md) |

`SKILL.md` states the rules the first five justify: put the invariant in `init`,
let the wire shape be primitives, read a `!!` as a modeling error, answer the Java
question with `data class` whenever the answer isn't a confident no, and ask of
every method added to an existing type whether that type's invariant still holds
on the paths the change added. Below is why each rule exists, which is what you
need when you are deciding whether to break one.

## 1. Deserialization: the factory is not on the path

**No deserializer calls your factory.** Not `PartySize.of`, not any other
companion function — a deserializer's job is to reconstruct an object from a
payload, and it reaches for constructors. Any check living only in a companion
factory is therefore absent at a REST or message boundary, which is the exact
path the discipline exists to protect.

`init` is the stronger guard because it sits on every construction path the
*language* provides. Whether your library takes one of those paths is a separate
question, and it is a question about your library at your version — see
[deserialization.md](deserialization.md), which carries the three to ask and a
measured answer for two libraries.

**The Kotlin half is not a question, and it is the trap.** Where every
constructor parameter has a default, Kotlin synthesises a no-arg constructor.
A library that finds one uses it: `init` runs against the defaults and *passes*,
and the payload's values are written into the fields afterwards. No exception is
raised anywhere, and the object that comes out violates its own invariant.

That is the one shape that breaks silently, and **the defaults create it, not
the library** — which is why no amount of choosing a better deserializer closes
it. The fix is structural: parse into a separate DTO with primitives only, and
run the factory yourself. Shape and trade-offs in
[deserialization.md](deserialization.md).

## 2. `copy()`: a private constructor is not a closed door

A `data class` generates `copy()` **public**, regardless of the primary
constructor's visibility. So `private constructor` plus a validating factory
leaves anyone holding one valid instance able to produce arbitrary others.
`ReservationTime` is the type in this skill that has to take the risk — its check
needs a `Clock` and the `OpeningHours`, so it cannot live in `init`:

```kotlin
// `valid` came out of ReservationTime.of — the only path that consults a clock.
// Drop the @ConsistentCopyVisibility that type carries, and this compiles:
val absurd = valid.copy(value = Instant.EPOCH)   // no factory, no clock, no complaint
```

**Put the invariant in `init` and `copy()` honours it too.** That fix needs no
flag and no annotation, holds on every Kotlin version, and is always the first
move.

It then owes the factory an answer, and **both obvious ones are wrong**:

- **Catching your own `require`** turns an exception into control flow, inside
  the one type that was supposed to make the exception unnecessary.
- **Restating the condition in the factory** leaves two copies of one rule with
  nothing keeping them in agreement — the failure mode is a third condition
  added to one copy and not the other.

Write the predicate once, in a private function that `init` and the factory both
call. The shape is in [worked-examples.md](worked-examples.md).

**Where the check genuinely cannot live in `init`** — it needs a clock, a floor
plan, a price list — the visibility gap has to be closed instead. Which flag,
which annotation, whether the compiler still requires either, and why the
module-wide flag beats the per-class one: [worked-examples.md](worked-examples.md).

## 3. The force-unwrap: `!!` is never the fix, it's the diagnostic

A `!!` on a factory's result is the modeling telling you it went wrong, and
there are exactly two ways it went wrong:

- **The value is internal-only**, so the factory should never have been nullable.
  An asserting constructor removes the nullability and the runtime check
  together — one piece of code failing fast for its caller and typing the value
  away for everyone above.
- **It really can arrive invalid**, so the `null` is a case that needs handling,
  and force-unwrapping deletes the case rather than answering it.

**This row is the one you settle mechanically.** `!!` is plain text, so a search
over the files your change touched finds every one, including those outside the
hunks you are reading. Read each as a design question.

Where a nullable genuinely has to fail fast, `requireNotNull`/`checkNotNull`
with a message is the tool. **`!!` is the same throw with the message deleted** —
the stack trace names a line, and nothing else.

**The quiet third case** is test setup. `PartySize.of(4)!!` in every fixture is
how `!!` enters a codebase and then normalizes, until nobody reads it as a
finding any more. Give the module a test factory that asserts with a message,
and the production rule survives contact with the test suite.

## 4. Java interop: the door `@JvmInline` leaves open

Kotlin's own
[KEEP for inline/value classes](https://github.com/Kotlin/KEEP/blob/master/proposals/inline-classes.md)
documents that Java can pass the underlying primitive straight into a function
taking a `value class`, **skipping the constructor and its `init` block** — the
proposal's own example annotates that call `// constructor or initialization
block wasn't called`. A non-`value` class has no such gap: its constructor
always runs.

**This is the one bypass with no fix inside the type.** `init` cannot close it,
because the constructor is what gets skipped. A factory cannot, because no Java
caller is invoking one. The only close is not to be a `value class` on that
signature — which is why `SKILL.md` answers "don't know" with `data class`, and
why the trigger above is `@JvmInline` on a type Java can reach rather than
`@JvmInline` on its own.

Where the type stays a `value class` because nothing Java-side reaches it, the
four cases that make it stop paying for itself anyway — a second property, life
inside a generic collection, reflective construction, only ever nullable — are
in [worked-examples.md](worked-examples.md).

## 5. The new path: an invariant that held until this change

The other four are properties of a type. This one is a property of a **diff**:
the type was fine, its check was in `init`, nobody touched it — and the change
added a way in that does not run it.

**The trigger is the type's age, not the method's shape.** A method added to a
type your change did not create is the row firing, and there are three shapes it
takes, all of them visible in the diff you are already reading:

- **A new factory** beside the existing one, written for a caller holding the
  value in a different shape — and reproducing the condition instead of calling
  the predicate the type already has. Now two copies of one rule.
- **A new mutator**: a `withX`, a builder step, a `val` promoted to `var`. It
  produces a state the constructor would have rejected, without going near the
  constructor.
- **A new `copy` site.** `copy()` was always public; what changed is that
  somebody is calling it now, with the field that carries the invariant among the
  named arguments.

Where the invariant is in `init` and the new path runs a constructor, reading it
settles the row. Where it lives in a factory, this row and `copy()` above are one
finding arriving twice — and the fix is the same one bypass 2 gives: move the
predicate somewhere both paths call it.

