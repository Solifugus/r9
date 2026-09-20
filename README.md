# Rachis9 (R9)

**A language for autonomous real-world systems — robots, vehicles, machines,
and the controllers that sense and act on the physical world.**

> **Status: design, not implementation.** There is no compiler. This
> repository is a design document, a tutorial, and example programs written
> to find out where the language cannot yet say what was meant. If you are
> looking for something to run, there is nothing here yet.

A *rachis* is a spine: the central axis connecting higher-level intention to
deterministic reaction and physical action. The numeral marks its
relationship with [RV-9](../rv9), the operating system that admits,
schedules, measures and enforces what R9 compiles — and suggests the other
thing the language is about, which is failing predictably.

---

## The idea

Three layers, with authority running **downward**:

```
            PROACTION      decide what should happen next
                |
            REACTION       given conditions, reach known states
                |
            REALTIME       do physical things correctly and on time
                |
              RV-9         admission, scheduling, devices, enforcement
                |
         PHYSICAL WORLD
```

Safety and timing constraints live at the bottom and constrain everything
above them. Intelligence and uncertain reasoning live at the top and are
never permitted to override the layer below. That direction is the whole
design.

## What it is trying to prevent

Two classes of mistake, by making them not expressible.

**Physical dimensions are part of the type system.** Temperatures,
percentages, durations and angles are values with units, not numbers with a
comment nearby. A metres/feet or radians/degrees confusion becomes a
compile error rather than a vehicle in a ditch.

```
const TARGET_DAY = 24C
const SOAK       = 2min
const MAX_RUN    = 20s
const TANK_LOW   = 10%
```

**Timing is declared, not hoped for.** A real-time component states its
period and its deadline, and the declaration is a contract the operating
system reads and enforces:

```
realtime CLIMATE every 500ms
    deadline 200ms
```

RV-9 takes that, together with a measured worst-case cost, and **refuses to
start the program** if admitting it would make any loop miss its deadline —
naming which loop and by how much. The guarantee is not that the code is
careful; it is that an unschedulable system does not run.

## Where it actually stands

Honest, per the design document's own markers:

| layer | state |
|---|---|
| **REALTIME** | relatively mature — timing, devices, bounded execution, faults |
| **REACTION** | well defined in shape; several constructs provisional |
| **PROACTION** | architecture settled — serialized authority, authority domains, a bounded log — and **no grammar** |

The concrete syntax of the language as a whole is given **by example rather
than by definition**. The design document marks every claim as *settled*,
*provisional*, *implementation question* or *open research*, and §42 lists
what remains open in each layer.

The examples are written specifically to find the gaps. `examples/grow_chamber.r9`
annotates every place the language could not express what was meant, and
collects them at the bottom of the file.

## Reading order

1. [`docs/tutorial.md`](docs/tutorial.md) — the language by example.
2. [`docs/autonomous_real_world_language_design.md`](docs/autonomous_real_world_language_design.md) —
   the full design, with what is settled and what is not.
3. [`examples/grow_chamber.r9`](examples/grow_chamber.r9) — REALTIME and
   REACTION, with the gaps marked.
4. [`examples/grow_chamber_proaction.r9`](examples/grow_chamber_proaction.r9) —
   what the deliberative layer would have decided.

## Its other half

R9 targets [**RV-9**](../rv9), which exists and runs. RV-9 publishes a
machine-readable target profile — the module ABI, every manifest tag and
whether the firmware enforces it, which system calls are real-time safe,
the fault codes and the limits — so that a compiler can read what the
system will actually promise rather than being told in prose.

That contract is the reason the two projects are separate and why neither
is much use without the other.

## Licence

Apache License 2.0 — see [LICENSE](LICENSE) and [NOTICE](NOTICE).
The name is not covered by that licence.
