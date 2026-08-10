# Adapters: taking a result out to an edge

A result leaving the domain. `SKILL.md` carries the rules; this file carries the
code. `cancelling` is a stage of the restaurant flow in
[worked-examples.md](worked-examples.md).

## Mapping a result out (`cancelling`)

A sealed result is the domain's vocabulary. A status code is HTTP's. The
translation between them belongs to the adapter, and nowhere else:

```kotlin
// In the HTTP adapter — the one place in the system that knows about status codes.
internal fun CancelReservationResult.toResponse(): ResponseEntity<Any> =
    when (this) {
        CancelReservationResult.Cancelled ->
            ResponseEntity.noContent().build()
        CancelReservationResult.NotFound ->
            ResponseEntity.notFound().build()
        CancelReservationResult.AlreadyStarted ->
            ResponseEntity.status(HttpStatus.CONFLICT)
                .body(Problem("The reservation has already started"))
        CancelReservationResult.Contended ->
            ResponseEntity.status(HttpStatus.CONFLICT)
                .header(HttpHeaders.RETRY_AFTER, "1")
                .body(Problem("Someone else changed this reservation — try again"))
    }
```

Now the same result type consumed by a queue worker instead:

```kotlin
// In the queue consumer — same result type, entirely different vocabulary.
internal fun CancelReservationResult.toAck(): Ack =
    when (this) {
        CancelReservationResult.Cancelled,
        CancelReservationResult.NotFound,
        CancelReservationResult.AlreadyStarted -> Ack.Done
        CancelReservationResult.Contended -> Ack.RequeueWithBackoff
    }
```

Put the two side by side and the rule argues for itself. `NotFound` is a 404
over HTTP but a perfectly successful `Done` on the queue — there's nothing to
retry and nothing to report to a caller who has already gone away. `Contended`
is a 409-with-`Retry-After` over HTTP but a requeue with backoff on the
worker. **No single status code is "the" translation of a variant**, which is
exactly why the translation can't live on the variant.

What that rules out, concretely:

- **No `@ResponseStatus` on a result variant, no `httpStatus` field in the
  sealed hierarchy, no `HttpStatus` in a use case's signature.** The adapter
  depends on the domain; the domain must not depend on the adapter. The moment
  a status code appears in the sealed type, the queue consumer above becomes
  impossible to write honestly.
- **No `else ->` branch.** The whole return on modeling outcomes as a sealed
  type is that adding a variant breaks compilation at every edge until someone
  decides what it means there. An `else -> 500` throws that away and converts
  every future outcome into an outage-shaped response.
- **`Contended` is never a 500.** A 500 tells the client "we are broken"; a
  conflict means "nothing is broken, try again." Getting this wrong makes
  ordinary contention look like an incident on every dashboard you own.

One design signal worth watching: if the mapping turns out to be mechanical —
every variant with exactly one obvious status, nothing else in the `when` —
the sealed type was probably designed backwards, from status codes inward,
rather than from the domain outward. That's the same smell as a result type
that renames another result type variant-for-variant. The domain's outcomes
should be nameable without knowing that HTTP exists.

## The same rule for any dependency you own

Persistence is the common case, not the only one. **Translate at the boundary
that owns the dependency** — an HTTP client, a parser, a payment SDK, a broker
library. The class that chose the dependency is the only one that can tell which
of its failures are business-reachable (the card was declined), which are
transient (the terminal timed out) and which are genuine bugs (the request was
malformed). Above that class, callers see your vocabulary and never the
library's; otherwise replacing the library becomes a change to every caller up
the chain.

## When the result type crosses a versioned boundary

Inside one Gradle build, a new variant breaking every `when` is the point — the
compiler routes you to each place that must decide. Once the sealed type is
published in an artifact that consumers compile against separately, or its
variant names go out over the wire, that same break lands on someone else's
schedule.

- **Don't add an `Unknown` or `Other` variant to spare yourself the break.**
  That's an `else ->` branch wearing a different hat: it converts every future
  outcome into a case consumers can't act on, permanently, from the first
  release.
- **The consumer owns the unknown case.** A client deserializing outcome names
  across a versioned boundary is the party that must decide what an
  unrecognised one means for it — usually "treat as retryable and alert", never
  "silently succeed".
- **Adding a variant is a breaking change unless consumers already handle the
  unknown case.** Version it as such rather than hoping. A `sealed interface`
  is a closed set by design; publishing one is a promise about that set.
