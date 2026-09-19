# Notes Toward a Book on RV-9 and R9

**Started 2026-09-19.** Not the book, and not design decisions — those belong
in `autonomous_real_world_language_design.md`. This is a place to catch
framings and worked examples while they are fresh, because the best of them
have arrived sideways, in conversations about something else.

---

## The creature

A slithering animal in the mud under shallow water. Whatever it senses ahead
is sensed moments later by the parts behind, so it learns the world as a
succession: this, then that, at this interval.

The frame earns its place because the machine architecture lands on the same
shape for the same reason — **time is serial, and parallelism is bandwidth as
we cross it**. Sensing, searching and predicting all happen at once;
committing to an act cannot, because each act changes the world the next one
is chosen against.

### Three horizons of memory

The clearest first-chapter explanation of the layers we have found, and
better than the layer names:

| horizon | in the creature | in R9 |
| --- | --- | --- |
| reflex, no memory | flinch from the sharp thing | REACTION re-observes reality and remembers nothing |
| working, medium | where I was going just now | PROACTION's in-memory state, lost on a power cut |
| persistent, broad | this stretch of bank is dangerous | the checkpoint and the log |

The layers exist because those horizons need different guarantees, not
because three layers seemed tidy.

### Growing senses and limbs

Evolution adds a sense without rewiring the old ones, which is what §45.1
says of hardware: *the set of things that could exist is static; the set
present varies, and presence is published reality*. New actuators become new
authority domains. The creature that grows a fin acquires a thing that can
contend with the tail for attention.

That suggests the shape of the book: **one machine across every chapter**,
gaining a sense or a limb per chapter, with the language growing in the order
the design actually grew.

- a bump switch, a reflex and a failsafe — REALTIME, and why a dying loop
  cannot be trusted to clean up after itself;
- a temperature sense — a second component, publication, and coherence;
- a state worth being in — predicates over reality, and why not labels;
- a goal — transitions, planning, and requests that can fail with reasons;
- memory of yesterday — the log, relative-time views, curtailment;
- a limb — authority domains, contention, and `in parallel`;
- a second machine — federation, and why authority does not cross machines.

### Two cautions

**The biology is a fable, not a history.** A sense ordering of touch, pain,
temperature, smell, then eyes is plausible-sounding and probably wrong:
chemoreception and mechanoreception are extremely old — single cells do
chemotaxis — opsins predate bilaterians, and most of that list predates
vertebrates entirely. Either present it explicitly as an imagined creature or
check it against a real source. The engineering argument does not depend on
the biology, which is exactly why it must not be staked on it.

**The metaphor invites conclusions the design refuses.** Reflexes, working
memory and durable belief look like a theory of mind, and §34.1 says
PROACTION encodes none. The creature explains *why the layers exist*, never
what a mind is.

---

## What this book could do that others cannot

**Show the measurements.** RV-9 has real numbers on real hardware: jitter at
1 kHz, the cost of a publication warm and cold, what an SSH session costs in
kilobytes, what happens when memory runs out and a control loop is admitted
anyway. A chapter that puts a loop's declared bound beside its worst measured
response would be unusual and hard to argue with.

**Keep the honesty markers.** The design marks every claim settled,
provisional, implementation question, or open research. A technical book that
says which parts are decided, which are guesses, and which are unsolved would
be rarer than it should be.

**Show the corrections.** Several of the best decisions came from catching the
document promising something the machine did not do — watches firing on change
when the mechanism wakes on publication, a watcher that would have stayed
blind until something moved. Those are better teaching than the tidy result.

---

## Worked examples on hand

- **`examples/grow_chamber.r9`** — REALTIME and REACTION halves, and thirteen
  recorded failures to say what was meant. The gap list is teaching material
  in its own right.
- **§40's shuttle cart** — the smallest complete program that shows planning,
  preemption and a hand-over ordered so that something always holds the load.
- **RV-9's boot suites** — tests that run on the hardware at every boot, which
  is where the measurements would come from.

---

## Open questions about the book itself

- One book or two? RV-9 is an operating system that admits contracts; R9 is a
  language that describes them. They are one story told from two ends, and the
  co-design is the interesting part — which argues for one book with two
  halves.
- Who is the reader: an embedded engineer who wants the machine to stay up, or
  someone building autonomy who has never met a deadline miss? The two want
  opposite orders of presentation.
- How much of the design document survives into it? The section-by-section
  structure is historical, which is good for the authors and bad for a reader
  meeting it cold.
