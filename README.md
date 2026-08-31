> [!CAUTION]
> Read this skill in full before using it — do not rely on a summary. It was created for educational purposes and for educational experimentation; the author takes no responsibility — in every case, in particular in a professional or work environment, but equally so for private, personal use — for its use or misuse. Each time you accept or act on a decision it proposes, you are solely responsible — in every case, in particular in a professional or work environment, but equally so for private, personal use — for evaluating it, for understanding the context it applies to, and for the results your agent or language model produces. Be aware that a language model or agent can skip or ignore a skill's instructions entirely; nothing guarantees a skill was actually applied, so verify accordingly. Its content was substantially produced by various AI models, directed by the author — who supplied his own experience, deliberately devised the examples fed in as input, questioned and challenged them, and iterated together with the models across multiple sessions of conversation, exploration, and experimentation. It follows the Agent Skills format at [agentskills.io/specification](https://agentskills.io/specification).

# kotlin-domain-modeling

An Agent Skill for modeling domain data in Kotlin: types that cannot hold an
invalid value, failures that come back as data rather than as thrown exceptions,
and the case where those two meet — two callers deciding against the same state.

**One idea holds it together.** A check produces a *fact*, the fact needs a
**carrier**, and the carrier decides how long it stays true. Put the fact in the
type and nothing can make it false; put it in a result and it can expire, but
visibly; write it down nowhere and nothing can rest on it. The three kinds of
invalid fall out of that as three lifetimes of one fact — and the third of them
is contention, which is why this skill treats locking as part of modeling rather
than as an appendix to it.

## What is in it

`SKILL.md` carries the decisions — the carrier and the three kinds of invalid,
the ladder (*type it away*, *hand it back*, *fail fast*), *read–decide–write*
and the three anomalies, the type-level shapes, and where an invariant is
enforced. Everything longer sits in `references/`, cut by **what triggers a
load**: a file exists only where nothing else's pointer fires on the same
moment.

| Reference | Load when |
|---|---|
| [`review.md`](kotlin-domain-modeling/references/review.md) | Reviewing Kotlin rather than writing it — three passes |
| [`bypasses.md`](kotlin-domain-modeling/references/bypasses.md) | Before calling a change done: seven paths that reach an object without running its check |
| [`choosing-the-rung.md`](kotlin-domain-modeling/references/choosing-the-rung.md) | The rung is not obvious: both mistakes worked through in code |
| [`worked-examples.md`](kotlin-domain-modeling/references/worked-examples.md) | The shapes in full, on one restaurant flow |
| [`deserialization.md`](kotlin-domain-modeling/references/deserialization.md) | A value arrives from outside |
| [`adapters.md`](kotlin-domain-modeling/references/adapters.md) | A result leaves the domain for HTTP or a queue |
| [`contention-code.md`](kotlin-domain-modeling/references/contention-code.md) | Two callers decide against the same state: the aggregate, and both strategies |
| [`contention-operations.md`](kotlin-domain-modeling/references/contention-operations.md) | Before writing that against a real database |

## If you edit this

**Keep the vocabulary verbatim.** Each word replaced the same idea spelled out
at several sites, and spelling one back out is how that comes undone.

| Word | Means |
|---|---|
| **carrier** | what holds the fact a check produced: the value, a result, or nothing |
| *type it away* / *hand it back* / *fail fast* | the decision order, named rather than numbered |
| ***read–decide–write*** | the shape of any rule about state you do not own, and the interval that makes the fact expire |
| **bypass** | a path reaching an object without running the check written on it |
| *re-decide* | what a retry does: back to the read, not back to the write |
| **detection** / **policy** | *was this contention?*, which is yours; *how do we retry?*, which is a library's |

**A named type has one home.** The file that spells out `PartySize`'s fields is
the only one that spells them out; every other file calls it by name and links.

**Examples live on one restaurant flow**, its stages named for what happens
rather than numbered: `booking`, `party`, `seating`, `ordering`, `kitchen`,
`cancelling`, `scheduling`. A new example goes on a stage rather than inventing
a domain, and the table in `references/worked-examples.md` is the index.

**Every reference stays linked directly from `SKILL.md`.** A file reached only
through another reference is the nesting Anthropic warns about, where Claude
previews with `head -100` and reads it incompletely.

## Licence

MIT — see [LICENSE](LICENSE). Disclaimer and authorship are in
[`kotlin-domain-modeling/NOTICE.md`](kotlin-domain-modeling/NOTICE.md).
