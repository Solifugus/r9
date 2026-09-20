# Rachis9 (R9)

**A language for autonomous real-world systems — robots, vehicles, machines,
and the controllers that sense and act on the physical world.**

> **Status: design, not implementation.** There is no compiler. This
> repository is a design document, a tutorial, and example programs written
> to find out where the language cannot yet say what was meant. If you are
> looking for something to run, there is nothing here yet.

A *rachis* is a spine: the central axis connecting higher-level intention to
deterministic reaction and physical action. The numeral marks its
relationship with [RV-9](https://github.com/Solifugus/rv9), the operating system that admits,
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

**Authority runs downward, and is enforced rather than promised.** Whatever
the deliberative layer turns out to be — hand-written logic, a search, a
learned policy, a language model — it cannot drive an actuator. It requests
states; the reactive layer decides whether that is legal *now*; the real-time
layer keeps its own limits and failsafes whatever it is told. RV-9 enforces
this with mechanisms rather than good intentions: a publication cell has
exactly one writer, a device can be claimed exclusively, and a program that
asks for more than the machine can promise is refused before it starts.

```
transition SAFE -> AT(target)

    require BATTERY.charge > 20%
    require target >= TRACK_START and target <= TRACK_END
```

A transition whose `require` is false is not an edge the planner may use, so
a safety condition that must keep holding is expressed as a fact about the
world rather than as a lock. The intelligence is told exactly which
requirement blocked it.

## States describe reality, not labels

```
const STILL = 0.01m/s

state SAFE
    abs(DRIVE.speed) < STILL
    BRAKE.engaged
    DRIVE.powered == false
end

watch BATTERY.charge within 1s

    if BATTERY.charge < CRITICAL
        transition SAFE
    end

else
    transition SAFE          # the battery monitor has gone silent
end
```

A state is true when reality satisfies it, never because something assigned
the name. `transition SAFE` asks for a state and lets the planner find a
legal path from whatever holds now — and if that attempt is preempted
halfway, planning begins again from what is observed, because there is no
stored label to go stale.

The `within ... else` on a `watch` is how a program notices that a
publication has **stopped**. A system that only reacts to change never sees
silence, which is what an unplugged sensor looks like.

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
   what the deliberative layer would have decided, and the thirteen things it
   could not say.
5. [`docs/book-notes.md`](docs/book-notes.md) — framings kept for a book that
   does not exist yet.

## Building the printable tutorial

The PDF is not tracked; regenerate it with:

```sh
pandoc -s docs/tutorial.md -o docs/tutorial.pdf \
  --pdf-engine=weasyprint --css docs/print.css \
  --metadata title="Rachis9 — A Tutorial"
```

## If you want to help

The most useful thing anyone can do at this stage is **write a program in it**
and record every place the language could not say what was meant. That is how
the examples here were made, and it has found more than reading ever has — a
component with nowhere to keep state between activations, temperatures that
subtract wrongly, a scheduler in a language with no calendar.

## Its other half

R9 targets [**RV-9**](https://github.com/Solifugus/rv9), which exists and runs. RV-9 publishes a
machine-readable target profile — the module ABI, every manifest tag and
whether the firmware enforces it, which system calls are real-time safe,
the fault codes and the limits — so that a compiler can read what the
system will actually promise rather than being told in prose.

That contract is the reason the two projects are separate and why neither
is much use without the other.

## Licence

Apache License 2.0 — see [LICENSE](LICENSE) and [NOTICE](NOTICE).
The name is not covered by that licence.
