# Rachis9 (R9)

## Language Design for Autonomous Real-World Systems

**Status:** Draft design capture  
**Scope:** Real-time and reactive layers are conceptually stable; their RV-9 ABI and execution details remain open.  
**Open area:** Autonomy remains intentionally incomplete and requires further design.

---

## 1. Purpose

Rachis9, usually shortened to **R9**, is intended for **autonomous real-world systems**: robots, vehicles, machines, embedded controllers, industrial equipment, and other systems that sense and act upon the physical world.

The name reflects the language's architectural role. A rachis is a spine or central axis: R9 connects higher-level intention to deterministic reaction and physical action. The numeral identifies its close relationship with RV-9 and also suggests resilience: systems should fail predictably, recover where possible, and preserve safe physical behavior.

In this document:

- **Rachis9** or **R9** means the language, compiler, and its small target runtime;
- **RV-9** means the operating system that admits, schedules, measures, and enforces compiled programs.

The central design goal is to make difficult real-world behavior expressible with a **small number of powerful concepts**.

The language should:

- minimize programmer burden;
- use familiar mainstream syntax wherever possible;
- make physical dimensions and timing first-class enough to prevent mistakes;
- make deterministic physical behavior easy to express;
- keep intelligence and uncertain reasoning above deterministic execution;
- provide a natural path from low-level real-time control to reactive behavior and eventually autonomy;
- avoid unnecessary constructs when existing ones can express the same idea.

A recurring design principle is:

> **Do more with less.**

Reasonable behavior should come from the shortest form. Additional syntax should refine behavior, not create basic correctness from scratch.

---

# 2. Architectural Model

The system currently has two well-defined execution layers and one unfinished higher layer:

```text
    CONATUS / AUTONOMY
     goals / deliberation
     learning / reasoning
             |
             v
      R9 AUTONOMY INTERFACE
             |
             v
         R9 REACTIVE
   watch / state / transition
             |
             v
         R9 REAL-TIME
 timing / device use / expose
             |
             v
             RV-9
 admission / scheduling / devices
 enforcement / timing telemetry
             |
             v
       PHYSICAL WORLD
```

The current working interpretation is:

### Real-Time

> Execute physical operations with deterministic timing and publish coherent reality.

### Reactive

> Observe reality and deterministically transform it toward requested states.

### Autonomy

Provisional:

> Decide which states or outcomes should be pursued, especially when deterministic machinery cannot decide what to do next.

The autonomy layer may ultimately be provided by **Conatus**, rather than becoming a large independent language subsystem.

## 2.1 Relationship with RV-9

R9 and RV-9 are deliberately co-designed. They should be synergistic without collapsing into one another or duplicating responsibilities.

The governing contract is:

> **R9 describes what a program means, promises, and requires. RV-9 decides whether the physical machine can honor that contract, then enforces and measures it while the program runs.**

R9 is responsible for:

- language semantics and static checking;
- dimensional and representation analysis;
- dependency, state, and transition metadata;
- bounded-execution analysis where a bound can be established;
- stack and static-memory calculation or conservative estimation;
- generation of RV-9 modules and machine-readable resource manifests;
- generation of restricted declarative failsafe information.

RV-9 is responsible for:

- loading and validating modules and mandatory manifest entries;
- admission control against actual memory, timing, and device availability;
- scheduling periodic and event-released real-time work;
- enforcing device ownership and execution-class restrictions;
- measuring stack use, execution time, jitter, missed releases, and overruns;
- detecting process failure and applying failsafe behavior below the failed process.

These are target responsibilities, not claims that every mechanism is already implemented in RV-9.

The co-design contract flows in three directions:

1. **Target description: RV-9 to R9.** The RV-9 toolchain supplies a machine-readable target profile describing its ABI, module format, supported manifest entries, execution classes, and operations known to be real-time safe.
2. **Resource contract: R9 to RV-9.** The compiler emits modules with mandatory and advisory requirements, together with the strongest static evidence it can establish.
3. **Measured reality: RV-9 to R9.** At runtime RV-9 exposes admission results, timing measurements, resource use, and faults as ordinary observable state.

This closed loop lets the compiler reject known-invalid programs before deployment while allowing RV-9 to recheck claims against the actual machine and report when measured behavior disagrees.

The intended initial deployment model is a compiler-generated bundle:

| R9 concept | RV-9 representation |
| --- | --- |
| R9 program | A related bundle of RV-9 modules and metadata |
| Real-time component | An independently admitted and schedulable real-time program |
| Reactive layer | An ordinary bounded supervisor program |
| Published values and inputs | Fixed-size publication cells and mailboxes supplied by the R9 runtime contract |
| States and transitions | Compiler-generated tables used by the reactive planner |
| Failsafe declaration | Mandatory data interpreted by RV-9 or a trusted supervisor, not cleanup code in the failing process |

The exact bundle and publication ABI remain to be finalized. The architectural boundary does not: RV-9 should provide general enforcement mechanisms, not implement R9 keywords or autonomous planning, while R9 should use RV-9's native module, process, device, timing, and resource-contract mechanisms rather than recreate an operating system inside its runtime.

---

# 3. General Language Style

The language should look familiar to programmers coming from C, C++, Java, C#, Go, Python, or JavaScript.

Prefer conventional syntax unless the domain requires something better.

Examples:

```text
let x = 5:i32
let distance = 25m
let gain = 0.75:f32

x = x + 1

if temperature > 80C
    ...
else
    ...
end
```

Conventions currently favored:

- `()` for function calls
- `[]` for arrays/indexing
- `.` for member/component access
- `=` for assignment
- `let` for variable declaration
- `const` for immutable values
- `:` for explicit machine representation where needed

Example:

```text
let altitude = 12000m:f64
```

This says:

- physical meaning: distance
- unit: meters
- representation: `f64`

The physical meaning and machine representation are deliberately separate concepts.

---

# 4. Core Data Types

The type system should remain compact.

Likely core machine types:

```text
bool

i8
i16
i32
i64

u8
u16
u32
u64

f32
f64
f128      // target-dependent

fixed[a,b]

enum
record

type[n]   // fixed-size arrays
```

Large dynamic runtime structures are not required for the real-time layer.

Hard real-time code should avoid constructs that require unpredictable allocation or execution time.

Examples of things that should generally not be part of hard real-time execution:

- garbage collection
- arbitrary heap allocation
- unbounded containers
- reflection
- exceptions
- unconstrained recursive structures
- allocating closures

Fixed buffers are fine:

```text
u8[32]
char[64]
```

---

# 5. Physical Units and Dimensional Analysis

The language uses **SI / metric units only**.

Examples:

```text
12mm
25cm
3m
4km

5ms
2s

12V
4A
50N
20Nm
300K
```

Compound quantities may be written naturally:

```text
10m/s
9.81m/s2
```

The compiler performs dimensional algebra.

Example:

```text
let distance = 100m
let elapsed = 20s
let speed = distance / elapsed
```

The compiler infers that `speed` has dimension:

```text
length / time
```

Likewise:

```text
let speed = 10m/s
let elapsed = 2s
let distance = speed * elapsed
```

Similarly, electrical relations work through dimensional algebra without special-casing Ohm's law:

```text
let voltage = 12V
let resistance = 6Ohm
let current = voltage / resistance
```

The compiler can infer that the result has the dimension of electrical current.

The compiler does **not** need to know the physics equation by name. It only needs consistent dimensional rules.

### Important distinction

Physical dimension describes **meaning**.

Machine type describes **representation**.

Structure describes **shape**.

For example, speed and velocity share the same physical dimension:

```text
length / time
```

but may differ structurally:

- speed: scalar
- velocity: vector

Example:

```text
let position = [10m, 5m, 2m]
let previous = [9m, 5m, 2m]
let dt = 100ms

let velocity = (position - previous) / dt
```

The compiler can infer a three-element vector with the physical dimension `length / time`.

---

# 6. Type Inference

Type inference should reduce programmer burden.

The compiler should infer representation when semantic or contextual information makes the choice sensible.

Examples:

```text
let distance = 25m
let delay = 10ms
let throttle = 40%
```

When the programmer needs to specify representation:

```text
let distance = 25m:f32
let delay = 10ms:u32
let throttle = 40%:u8
```

For completely untyped literals with no semantic or contextual information, explicit representation may be required or governed by simple language defaults.

The guiding principle is:

> **Infer from semantic or contextual information. Do not invent representation when there is none.**

---

# 7. Overflow

A useful distinction exists between machine integers and physical quantities.

Plain machine integers may use well-defined wrapping semantics:

```text
255:u8 + 1
```

can become:

```text
0:u8
```

Physical quantities should probably **not silently wrap**, because that can turn a physical failure into plausible-looking bad data.

Example:

```text
let distance = ...
```

If the underlying representation cannot contain the result, the system should fault or otherwise make the failure explicit.

This remains an implementation detail to finalize, but the design direction is:

- raw machine arithmetic may wrap;
- physical quantities should favor safety over silent wraparound.

---

# 8. Declaration and Initialization

Preferred declaration syntax:

```text
let x = 5
let x = 5:i32
let distance = 25m
let gain = 0.75:f32
```

`let` introduces ordinary mutable storage.

`const` introduces immutable storage.

Declarations may be **lexically hoisted**, meaning the compiler knows the binding throughout its scope, but initialization must occur where written.

Example:

```text
do_something()
let x = read_sensor()
```

The name `x` may be known to the compiler throughout the scope, but its initializer is not moved upward.

Use before initialization should be rejected.

---

# 9. General Control Flow

Ordinary control flow should remain ordinary.

```text
if condition
    ...
else if other_condition
    ...
else
    ...
end
```

Loops:

```text
for ...
    ...
end
```

```text
while condition
    ...
end
```

Truly intentional infinite execution should be visually explicit:

```text
loop
    ...
end
```

This avoids accidentally turning an ordinary `while` into an unbounded real-time operation.

---

# 10. `await`

`await` is a **general language construct**, not specifically a real-time feature.

Example:

```text
await imu.ready within 100us
else
    fault IMU_TIMEOUT
end
```

Reactive example:

```text
await MOTOR_CONTROL.speed == 0m/s within 5s
else
    transition EMERGENCY_STOP
end
```

Semantics:

- if the awaited condition becomes true, execution continues;
- if the `within` interval expires, the `else` block runs;
- if no `within` is supplied, the wait may be indefinite where the execution context permits it.

Hard real-time contexts may place stronger restrictions on indefinite waiting.

---

# 11. REAL-TIME LAYER

## 11.1 Real-Time Components

The preferred terminology is:

- **real-time block**: syntax
- **real-time component**: language semantics
- **real-time task**: possible implementation mapping

A real-time component is independently schedulable and may contain:

- private state
- inputs
- timing requirements
- device operations
- published outputs
- failsafe behavior

Example:

```text
realtime MOTOR_CONTROL every 1ms

    let speed = calculate_speed()
    let current = read_current()

    let error = target_speed - speed
    let output = pid(error)

    limit output to -75% .. 75%

    motor.left = output
    motor.right = output

    expose speed, current, output

end
```

---

# 12. Meaning of `realtime`

Declaring something `realtime` places it in the highest scheduling class, but that alone is not sufficient.

The construct represents a deterministic timing contract.

Example:

```text
realtime CONTROL every 1ms
    ...
end
```

This form schedules the component periodically. A real-time component may instead be released by a bounded hardware or device signal.

The simplest useful timing vocabulary is currently:

```text
realtime
every
on
minimum_interval
deadline
within
priority
```

`priority` should generally be optional.

Potential advanced concepts such as `budget` and `phase` may be added later only if justified.

---

# 13. Timing Semantics

### `every`

Defines scheduling period:

```text
realtime IMU_UPDATE every 5ms
```

### `on` and `minimum_interval`

Defines a component released by a hardware or device signal rather than a periodic clock:

```text
realtime LIMIT_SWITCH on gpio.rising
    minimum_interval 200us
    ...
end
```

`minimum_interval` is the shortest interval between releases that the component promises it can handle. It is the sporadic equivalent of a period and allows RV-9 to include the component in admission and scheduling decisions.

An event source that violates the declared interval is not silently throttled. RV-9 records the excess releases or coalescing and exposes them as timing state.

This is a real-time release mechanism, not a separate reactive event subsystem. Periodic and event-released components share the same component-body and completion semantics; only their release source differs.

### `deadline`

Defines when the operation must be complete.

```text
realtime MOTOR_CONTROL every 1ms
    deadline 800us
    ...
end
```

If omitted, a sensible default may be:

```text
deadline = period
```

### `priority`

Most programmers should not need to manually assign priority.

RV-9 should derive scheduling priority or deadline order from the complete admitted workload and its periods, deadlines, minimum intervals, and execution bounds. A faster period alone is not always enough to choose correctly.

Explicit priority remains an escape hatch:

```text
priority 10
```

### `within`

Defines a bounded wait or operation:

```text
read imu within 100us
else
    fault IMU_TIMEOUT
end
```

## 13.1 How RV-9 answers this

Recorded 2026-09-13, against RV-9 as built (RV-9 `docs/design.md` §33).

**Priority is derived, as asked.** No program chooses a number. At every admission RV-9 places the whole real-time workload using response-time analysis over each component's release interval (`every` or `minimum_interval`), `deadline` (default: the period), and worst-case execution bound. It moves components that are already running when a new one changes the placement, and it refuses a component with `UNSCHEDULABLE` when no placement lets every deadline be met. That refusal can happen even when the CPU is far from full: two components using under 4% each are refused if their sub-millisecond deadlines collide.

**There are two levels, not a ranking.** On the current hardware exactly one host priority is above the radio, so RV-9 offers *urgent* (above everything) and *routine* (above all non-real-time work, below the radio). Placement starts everything urgent and moves the least urgent component down only while something urgent would otherwise miss its deadline. Deadline order is therefore respected where it matters, and a single component always runs at the top.

**The compiler should emit an execution bound for every `realtime` component.** RV-9 uses the declared bound in the analysis. Without one it uses the worst execution time measured so far, which is a floor and not a bound, so placement and refusal are only as trustworthy as that number.

**Open for this document:**

- **The `priority` escape hatch** shown above is not implemented. With two levels it could only mean "always urgent" or "always routine". Is that still worth offering, or does it only let a program defeat the analysis?
- **Routine components run below the radio**, and their bounds do not include it. Should the language be able to say "this component must never be placed below the radio", as a constraint rather than a number?

---

# 14. Bounded Execution

Bounded execution does **not** primarily mean forcibly killing a task after a time limit.

It means the compiler/runtime can establish or trust an upper bound on the amount of work that a real-time operation may perform before completion or yielding.

Bounded:

```text
for i = 0 to 9
    do_something()
end
```

Potentially unbounded:

```text
while sensor.notReady
    do_something()
end
```

Better:

```text
while sensor.busy within 100us
    ...
else
    fault SENSOR_TIMEOUT
end
```

or:

```text
await sensor.ready within 100us
else
    fault SENSOR_TIMEOUT
end
```

The design should make unbounded execution deliberate rather than accidental.

R9 should distinguish three kinds of bound:

- **established** — derived by the compiler from the call graph, loop bounds, representations, and operations whose target latency is known;
- **declared** — supplied as a requirement or claim when static establishment is not possible;
- **observed** — measured by RV-9 while the component runs.

The generated resource contract must not present a declared or observed value as a statically established guarantee. RV-9 can admit work against a timing claim, measure reality, and report or fault when execution violates it.

---

# 15. Real-Time Failure and Termination

Arbitrarily killing a physical control task may be unsafe.

The language should distinguish orderly termination from abnormal failure.

Potential forms:

```text
on stop MOTOR_CONTROL
    motor.left = 0%
    motor.right = 0%
    brake = engaged
end
```

and:

```text
failsafe MOTOR_CONTROL
    motor.left = 0%
    motor.right = 0%
    brake = engaged
end
```

`on stop` is cooperative cleanup executed while the component remains healthy enough to run it.

`failsafe` has a stronger and deliberately restricted meaning. It must compile to declarative safe-state operations that RV-9 or a trusted supervisor can perform without executing code in the failed component. A failsafe may name owned devices and constant safe configurations, but may not allocate, wait, call arbitrary functions, or depend on the failed component's private state.

A missed timing contract should cause predictable fault handling rather than blindly terminating execution in a way that leaves actuators undefined.

Principle:

> **A real-time construct must never leave the physical world undefined merely because execution stopped.**

## 15.1 `fault`

`fault` appears throughout this document — in `await`, in bounded waits, in transition bodies — and is defined here.

```text
fault NAME
```

`fault` stops the smallest enclosing construct that has a declared outcome, leaves the physical world in that construct's declared safe state, and publishes the reason.

It is **not** an exception. There is no unwinding, no handler, no propagation up a call chain, and no `try`/`catch` (§39). Execution stops where the statement is written.

Three things happen, in this order:

1. **The physical world is made defined.** If the enclosing construct is a real-time component, its declared `failsafe` is applied. This is not code in the component — the component has already stopped. It is the runtime writing declared constants to devices the component owned.
2. **The fault becomes ordinary published state.** The component publishes that it has faulted, and with what name.
3. **The construct does not run again.** A faulted real-time component is not rescheduled and its release source is stopped. Without this, the next period would immediately overwrite the safe state that step 1 just established.

The ordering of 1 before 2 is a requirement, not an implementation detail. An observer must never see a fault before the actuators are safe, because an observer may react to it by commanding something.

## 15.2 One keyword, two contexts

The same statement means the same thing in a transition body:

```text
transition MOVING -> STOPPED
    MOTOR_CONTROL.target_speed = 0m/s
    await MOTOR_CONTROL.speed == 0m/s within 5s
    else
        fault STOP_TIMEOUT
    end
end
```

Here the enclosing construct is the transition, whose declared outcome is defined by §29. `fault STOP_TIMEOUT` ends it with:

```text
attempt.status = FAILED
attempt.reason = STOP_TIMEOUT
```

Step 1 is empty because a transition owns no devices; steps 2 and 3 are §29's existing semantics. Nothing new is introduced — the keyword names an outcome the reactive layer already had.

The intent the programmer expresses is identical in both places: *I have detected that I cannot continue correctly; do the defined thing.* Only the consequence differs, and the compiler knows the context.

## 15.3 Faults nobody wrote

`fault NAME` is the *programmed* fault. Programs cannot anticipate every failure, so the mechanism above must exist regardless — and everything that stops a real-time component abnormally uses it. The keyword names an instance of a mechanism, rather than introducing one:

| origin | named by | reason |
| --- | --- | --- |
| `fault NAME` | the program | `NAME` |
| deadline missed | RV-9 | `DEADLINE` |
| stack exhausted | RV-9 | `STACK` |
| physical quantity overflow (§7) | R9 runtime | `OVERFLOW` |
| device failure | RV-9 | `DEVICE` |
| release-interval violation (§13) | RV-9 | reported as timing state, not a fault |

The last row is deliberate. An event source arriving too fast is a fault of the *world*, not of the component, and §13 already requires it be exposed as timing state rather than silently throttled. Faulting the component would remove the only thing still controlling the machine.

## 15.4 `fault` never runs `on stop`

`on stop` is cooperative cleanup for orderly termination, executed while the component is healthy. A faulted component is by definition not healthy enough to be trusted with code, so `fault` does not run it. This is the tempting mistake — *surely there is time for a little cleanup* — and the restriction in §15 on what a failsafe may contain exists precisely because there is not.

Orderly stop runs `on stop`. Failure applies `failsafe`. Nothing runs both.

## 15.5 Recovery is not a keyword

Per §30, a faulted component is observable state, and responding to it is ordinary reactive behavior:

```text
watch MOTOR_CONTROL.faulted

    if MOTOR_CONTROL.faulted
        transition SAFE
    end

end
```

**Open:** returning a faulted component to service is a change to reality, and therefore belongs to a transition rather than to new vocabulary — but the exact means by which a transition body restarts a component is not yet settled. No keyword should be added for it until the shape is clear.

**Also open:** a transition awaiting a value published by a component that has since faulted should fail immediately with `COMPONENT_FAILED` (§29 already reserves the reason) rather than waiting out its `within` interval and reporting a misleading timeout.

## 15.6 How RV-9 answers this

Recorded 2026-09-13, against RV-9 as built (RV-9 `docs/design.md` §29).

**Deadline faults are settled.** A module declares what a miss means in its manifest, as tag `RV9_MTAG_ON_DEADLINE` (0x0010, u8): `report` (0, the default) counts the miss and carries on; `fault` (1) ends the component. A compiler for this language should emit `fault` for every `realtime` component, and **mandatory**, so that an older RV-9 refuses the module rather than running it under the lenient policy.

- The deadline is the declared `deadline_us`, or the period when none is declared.
- What is compared against it is the **response**: lateness at release plus execution, with lateness measured against the release schedule rather than the previous wakeup.
- A miss is detected when an activation completes late, or when a release passes with no activation at all. The span from declaring to the first wait is initialisation and is not judged.

§15.1's three steps happen in §15.1's order. The release source is stopped; the exit path applies the declared failsafes; only then does the process table record `fault = DEADLINE` and exit status `-RV9_PE_DEADLINE`. The component's code is not called again — the process ends inside `rt_wait`, which never returns to it — so §15.4 holds by construction.

`STACK` was already in place: the kernel detects a stack overflow at a switch and applies the failsafe the same way.

**Stopping from outside** exists as `signal` (a request, `RV9_SIG_STOP` — what `on stop` would hang off) and `kill` (not a request). `kill` is not a fault in this document's sense — it is an operator's act — but it applies failsafes like every other exit, and records `killed` rather than a reason name.

**A body that never reaches its wait** is handled too (RV-9 `docs/design.md` §30). A watchdog flags a component still inside an activation after its deadline has passed, when a miss is fatal, and records `DEADLINE`. A component that holds the CPU for 250 ms without waiting is flagged whatever its policy, and records `RUNAWAY`.

Either is stopped from outside, and only at an instant when it is running its own compiled code. Generated code reaches the system only through calls that return, so at such an instant it holds nothing. The failsafe and the published reason then follow in §15.1's order.

**For this document to decide:** `RUNAWAY` is not a row in the §15.3 table. It is RV-9's, the compiler emits nothing for it, and it is arguably a case of `DEADLINE`. It is distinct so that "late" and "stopped responding" stay separate facts. Whether R9 names it, folds it into `DEADLINE`, or treats it as an operating-system fact outside the language is open.

**The fault is published in the component's own cell** (RV-9 `docs/design.md` §31), which is what `watch MOTOR_CONTROL.faulted` reads. The cell keeps the last value and stamp the component published, and carries the reason beside them; the publication sequence advances by one, so a watcher blocked on the cell wakes. Opening the cell to publish again clears it, which is how a restarted component becomes healthy without new vocabulary. `COMPONENT_FAILED` (§15.5) now has something to test: an `await` on a cell whose writer has faulted can see so immediately rather than waiting out its `within`.

**Publications are declared** rather than agreed by convention: `publishes` reserves a cell for a component at admission, so a second component publishing the same thing is refused before it starts, and `watches` is refused when nothing on the machine could ever publish that name. A compiler should emit both from `expose` and from `watch`.

---

# 16. Multiple Real-Time Components

A program may contain many independent real-time components:

```text
realtime MOTOR_CONTROL every 1ms
    ...
end

realtime IMU_UPDATE every 5ms
    ...
end

realtime RANGEFINDER every 20ms
    ...
end

realtime TELEMETRY every 100ms
    ...
end
```

These are language-level scheduling obligations.

They are not required to map one-for-one to conventional operating-system threads.

The compiler and operating system can cooperate directly:

- compiler knows timing requirements;
- compiler knows bounded waits and operations;
- OS scheduler knows actual runtime execution;
- runtime can report missed deadlines and other faults.

This design is intended to cooperate directly with RV-9 rather than sit on top of a generic RTOS abstraction.

## 16.1 Compiler-Generated RV-9 Resource Contract

Each independently admitted generated module should carry a machine-readable manifest containing the requirements applicable to it, including:

- execution class;
- stack and per-instance static storage;
- heap ceiling, normally zero for real-time components;
- period or minimum inter-arrival time;
- deadline and worst-case execution claim;
- required devices and shared or exclusive ownership;
- required capabilities;
- failsafe configuration;
- R9 compiler ABI and runtime version.

Requirements that would make execution unsafe if ignored must be emitted as mandatory manifest entries. RV-9 must refuse a module whose mandatory requirement it does not understand.

Initialization and admitted execution are separate phases. A component may open and configure devices and prepare fixed buffers before entering its timing contract. Once admitted, its real-time path may use only operations that RV-9 identifies as bounded and real-time safe.

---

# 17. Component Scope

Real-time components have lexical scope.

Values inside the component are private unless deliberately published.

Example:

```text
realtime MOTOR_CONTROL every 1ms

    let speed = encoder.speed
    let error = target_speed - speed
    let output = pid(error)

    expose speed, error, output

end
```

External code accesses published values through dot notation:

```text
MOTOR_CONTROL.speed
MOTOR_CONTROL.error
MOTOR_CONTROL.output
```

This avoids a global-variable model and gives the compiler explicit dependency information.

---

# 18. `expose`

`expose` is not merely an access modifier.

It is a **publication and synchronization operation**.

Example:

```text
expose speed
```

Before publication, outside observers continue to see the previous published value.

When `expose` occurs, the current value becomes externally visible.

Multiple values may be published atomically:

```text
expose speed, current, temperature
```

Reactive observers should see the newly published set as a coherent update, not a mixture of old and new values.

This solves a concurrency problem without requiring the programmer to use locks for ordinary publication.

### First publication

Before the first `expose`, a value is simply **not yet externally available**.

The design does not currently introduce an `unknown` data type.

The first publication establishes the baseline.

Subsequent changed publications may trigger reactive watchers.

### RV-9 publication contract

The R9 runtime representation of an exposed set should be fixed-size and preallocated. Each atomic publication should carry at least:

- a validity indication for first publication;
- a monotonically increasing publication sequence;
- a timestamp associated with the physical observation;
- the coherent set of exposed values.

The exact shared-memory or system-service ABI remains open. It must work with RV-9 process isolation and must not require allocation or blocking locks on a real-time path.

## 18.1 How RV-9 answers this

RV-9 has now implemented the contract above, and the answer settles the ABI question: **a publication is a device.**

```text
/pub0/MOTOR_CONTROL
```

A cell is a named, fixed-size piece of memory on a device that serves them. `expose speed, current, output` compiles to one `write` of one struct to one path. The publisher opens the cell before entering its timing contract; publishing thereafter is a memcpy and two stores.

Shared memory was rejected because it does not survive RV-9's process isolation, and a message queue because `watch` wants the current value rather than every value — a queue must either grow without bound or drop, and both are wrong answers to "what is the speed now".

Every publication carries exactly the four things §18 asks for:

```c
struct {
    uint32_t seq;        /* 0 = never published; then one per publication */
    uint32_t len;
    uint64_t stamp_us;   /* when the observation was made */
    /* the coherent set follows */
};
```

`seq == 0` is the validity indication, so first publication needs no `unknown` type. `stamp_us` is the time of the *observation*, not of the publication; a publisher that passes zero gets the time of the write.

Coherence across a set is a seqlock, and the asymmetry is deliberate: the writer never waits, never allocates and never takes a lock, so the whole cost of contention falls on the observer. That is what makes the real-time half of this contract keepable. An observer that cannot take a snapshot between two publications is told so and counted, rather than made to spin.

One writer per cell is enforced; readers are unlimited. Cells are preallocated at boot and are **not** freed when their publisher exits, so the last thing a stopped component said, and when it said it, remains readable — which is what §15's fault handling needs on the other side.

`watch` (§21) is served by a blocking wait against the sequence rather than against an edge, so a publication that lands while nobody is waiting is still seen, and several arriving together coalesce into one wakeup.

**Inputs (§19) need nothing further.** They are this same object with the ownership reversed: the supervisor is the single writer and the real-time component the observer.

**Still open on the RV-9 side.** A cell is created by whoever first opens it for writing, so two components agreeing on a name do so by convention. Declaring published cells in the module manifest — the way required devices already are — would let admission refuse a watcher naming a component that will never publish, and is the natural next step.

---

# 19. Inputs

Reactive or higher-level code should not arbitrarily mutate a real-time component's private state.

Writable inputs should be explicit.

Conceptually:

```text
realtime MOTOR_CONTROL every 1ms
    input target_speed

    ...
    expose speed
end
```

External code may then write:

```text
MOTOR_CONTROL.target_speed = 2m/s
```

but cannot directly assign to private or published output state.

Exact input syntax remains an implementation detail, but the boundary is important.

---

# 20. REACTIVE LAYER

The reactive layer deliberately introduces very little new vocabulary.

Its nucleus is:

```text
watch
state
transition
```

Everything else should mostly come from ordinary language constructs such as:

```text
let
const
if
else
for
while
await
within
```

There is currently **no separate reactive event subsystem**. Hardware events may still act as release sources for real-time components.

There is currently **no `heal` keyword**.

There is currently **no `recover` keyword**.

There is currently **no runtime proof system**.

---

# 21. `watch`

`watch` reacts to explicit published references.

Example:

```text
watch MOTOR_CONTROL.speed, BATTERY.voltage

    if MOTOR_CONTROL.speed > MAX_SPEED
        transition SLOW
    end

    if BATTERY.voltage < MIN_VOLTAGE
        transition SAFE
    end

end
```

A watch observes **variables/references, not arbitrary conditions**.

Preferred:

```text
watch MOTOR_CONTROL.temperature
    if MOTOR_CONTROL.temperature > 80C
        ...
    end
end
```

Not preferred:

```text
watch MOTOR_CONTROL.temperature > 80C
```

The explicit-reference form gives the compiler/runtime a dependency graph.

This enables efficient behavior:

- reevaluate only when relevant values change;
- avoid continuous polling;
- coalesce related changes;
- short-circuit work when dependencies did not change.

This matches the design principle of doing more with less.

---

# 22. No Separate Reactive Event Block

A separate reactive event construct currently appears unnecessary.

Changes to published values already provide a natural trigger mechanism.

Failure information can also be published or represented as ordinary values and observed through `watch`.

Therefore:

> **If ordinary values plus `watch` can express it cleanly, do not add an event subsystem.**

---

# 23. Reactive State

A `state` is **not merely a stored enum label**.

A state is a predicate describing reality.

Example:

```text
state SAFE

    MOTOR_CONTROL.speed == 0m/s
    MOTOR_CONTROL.power == 0%
    BRAKE.engaged == true

end
```

This means:

> SAFE is true when these conditions are true.

The state does not become true merely because software assigns the word `SAFE`.

If reality already satisfies the predicate, the system is already in that state.

This is intentionally closer to predicate-based planning than to a conventional finite-state machine.

Simpler states are still possible:

```text
state IDLE
    mode == IDLE
end
```

---

# 24. State Semantics

The design explicitly chooses:

> **State conditions define the state.**

They are not merely preconditions for entering the state.

Thus:

```text
state PARKED
    MOTOR_CONTROL.speed == 0m/s
    BRAKE.engaged
end
```

defines what `PARKED` means.

This allows the transition planner to reason about desired reality rather than symbolic labels alone.

---

# 25. Transition Definitions

A transition definition tells the system how one known transformation can occur.

Example:

```text
transition MOVING -> STOPPED

    MOTOR_CONTROL.target_speed = 0m/s

    await MOTOR_CONTROL.speed == 0m/s within 5s
    else
        fault STOP_TIMEOUT
    end

end
```

Another:

```text
transition STOPPED -> PARKED

    BRAKE.engaged = true

end
```

Another:

```text
transition PARKED -> SAFE

    MOTOR_CONTROL.enabled = false

end
```

These definitions form a state-transition graph.

Conceptually:

```text
MOVING
   |
   v
STOPPED
   |
   v
PARKED
   |
   v
 SAFE
```

---

# 26. Transition Requests

A transition request asks the system to make a target state true.

Example:

```text
transition SAFE
```

This does **not** mean:

```text
current_state = SAFE
```

It means:

> Given current reality, known transitions, and legal constraints, determine a valid transformation plan that results in the predicate `SAFE` becoming true.

If the current state is `MOVING`, the planner might determine:

```text
MOVING -> STOPPED -> PARKED -> SAFE
```

and execute those transformations.

---

# 27. Transition Planning

The preferred conceptual term is:

> **transition planning**

rather than merely pathfinding.

Pathfinding is one possible implementation technique.

The semantic problem is broader:

> Given current state, desired state, available transformations, and constraints, construct a valid sequence of operations that transforms the current world into the desired world.

This is similar in spirit to:

- classical AI state-space planning;
- STRIPS-style planning;
- chess search;
- SQL query planning.

The SQL analogy is particularly useful.

SQL says what result is desired and allows the database to determine how to obtain it.

Likewise:

```text
transition SAFE
```

states a desired result while the transition planner determines the execution path.

---

# 28. Alternative Transition Paths

Multiple legal paths may exist.

Example:

```text
transition MOVING -> STOPPED
    ...
end

transition MOVING -> EMERGENCY_STOPPED
    ...
end

transition EMERGENCY_STOPPED -> SAFE
    ...
end
```

The planner may choose among them.

A future design may include transition cost:

```text
cost 10
```

but this should **not** be added until a real need appears.

A simple shortest-valid-path or otherwise deterministic rule may be enough initially.

Do not add optimization syntax prematurely.

---

# 29. Transition Results

Transition attempts should produce observable results.

Example:

```text
let attempt = transition SAFE
```

The result may expose fields such as:

```text
attempt.status
attempt.reason
```

Possible status values:

```text
PLANNING
RUNNING
SUCCEEDED
FAILED
```

Possible failure reasons:

```text
NO_PATH
TIMEOUT
PRECONDITION_FAILED
COMPONENT_FAILED
CONSTRAINT_VIOLATION
```

These may simply be enums.

The important part is that transition outcome is **ordinary observable state**.

Therefore it integrates naturally with `watch`.

Example:

```text
watch attempt.status

    if attempt.status == FAILED
        transition EMERGENCY_SAFE
    end

end
```

This avoids inventing:

- transition-failed events;
- exception handlers;
- special recovery blocks.

---

# 30. Self-Healing

Self-healing does not currently need a language keyword.

It is simply reactive behavior expressed using `watch`.

Example:

```text
watch MOTOR_CONTROL.temperature

    if MOTOR_CONTROL.temperature > MAX_TEMP
        MOTOR_CONTROL.power_limit = 40%
        transition DEGRADED
    end

end
```

Sensor substitution:

```text
watch SENSOR_LEFT.available

    if SENSOR_LEFT.available == false
        NAVIGATION.sensor = SENSOR_RIGHT
        transition DEGRADED
    end

end
```

Failure of one recovery path can itself be watched:

```text
let recovery = transition SAFE

watch recovery.status

    if recovery.status == FAILED
        transition EMERGENCY_SAFE
    end

end
```

Thus:

> **Self-healing is a behavior built from observation and transition, not a separate subsystem.**

---

# 31. Reactive Planning vs Autonomy

A key architectural boundary has emerged.

### Reactive transition planning asks:

> Given a requested state, can I reach it using known legal transformations?

### Autonomy asks:

> Which state should I pursue, and why?

Example:

```text
let attempt = transition CHARGING_STATION
```

If it succeeds, the reactive system has completed the deterministic task.

If it returns:

```text
FAILED
reason = NO_PATH
```

the reactive layer has done its job.

The autonomy layer may then:

- choose another charger;
- change goals;
- gather information;
- wait;
- ask for assistance;
- create a new plan;
- abandon the original objective.

This keeps deterministic machinery deterministic.

---

# 32. Relationship to Classical AI

The reactive transition model resembles classical AI state-space planning.

A world may be represented by predicates such as:

```text
at(robot, roomA)
door_open(door1)
carrying(robot, box)
```

Actions have preconditions and effects.

The planner searches for a sequence of actions that changes the initial world into one satisfying the desired predicates.

The present language avoids exposing traditional predicate-logic syntax to the programmer.

Ordinary expressions already provide usable predicates:

```text
MOTOR_CONTROL.speed == 0m/s
BRAKE.engaged
BATTERY.voltage > 11V
```

The compiler can translate those expressions into an internal representation suitable for planning.

The design therefore borrows a powerful idea from classical AI without forcing the programmer to write an AI-specific logic language.

---

# 33. Proofing

Proofing is intentionally postponed.

It is **not a runtime concern**.

The likely model is closer to Ada/SPARK-style static reasoning, but potentially stronger because the compiler already understands:

- physical dimensions;
- timing contracts;
- bounded operations;
- published dependencies;
- state predicates;
- transition topology;
- transition reachability;
- actuator constraints;
- failure paths.

Potential future proof questions include:

```text
SAFE is always reachable from FAULT
```

```text
STOPPED implies MOTOR_CONTROL.power == 0%
```

```text
every path to SAFE disables propulsion
```

```text
CONTROL_TIMING failure always produces a safe actuator configuration
```

But no proof syntax should be designed until the execution model is complete.

---

# 34. AUTONOMY — CURRENTLY PROVISIONAL

The autonomy layer is deliberately not nailed down yet.

Initial brainstorming suggested it would need to:

- choose goals;
- decompose goals;
- select among alternatives;
- observe lower-level results;
- replan after failure;
- remember context;
- reason under uncertainty;
- gather information;
- coordinate objectives;
- decide when to stop or abandon a goal.

However, this list strongly resembles the existing **Conatus** architecture.

This raises an important possibility:

> The autonomy layer may not need to become a large new language subsystem.

Instead, the language may provide the deterministic substrate and interface that Conatus needs.

---

# 35. Possible Conatus Mapping

The mapping currently looks roughly like:

### Goals and goal choice

Conatus impetitive and aversive drives.

### Goal decomposition and deliberation

Contemplative Cursor (CC).

### Execution of known sequences

Real-Time Cursor (RTC), though deterministic portions may increasingly migrate into the reactive transition planner.

### Observation

Published component state and transition outcomes.

### Replanning after failure

CC.

### Learned successful behavior

Conatus sequence corpus.

### Substitution and novel approaches

Mind Splicer / analogical reasoning.

### Graded success

Drive satisfaction rather than a simple success/failure bit.

---

# 36. Potential Long-Term Architecture

```text
                 CONATUS
        goals / drives / values
        CC / deliberation
       learning / analogy
                 |
                 v
         AUTONOMY INTERFACE
                 |
                 v
          REACTIVE LAYER
      watch / state / transition
                 |
                 v
          REAL-TIME LAYER
      timing / devices / expose
                 |
                 v
                RV-9
     admission / scheduling
     devices / failsafe / telemetry
                 |
                 v
           PHYSICAL WORLD
```

This suggests a useful principle:

> **Intelligence proposes; deterministic machinery retains authority.**

The autonomy system may request:

```text
transition AT_LOADING_DOCK
```

The reactive layer decides whether and how that known state can be reached.

The real-time layer ensures the physical actions occur with deterministic timing and safety.

---

# 37. Behavioral Crystallization

There is a potentially important learning hierarchy:

```text
CC discovers
      ↓
RTC executes and validates
      ↓
stable behavior becomes known
      ↓
deterministic behavior can migrate into transition machinery
```

In other words:

> **Novel behavior starts high in the intelligent system and can move downward as it becomes understood, reliable, and deterministic.**

This may allow autonomous systems to become less computationally expensive and more predictable over time.

This concept remains exploratory but is strongly compatible with Conatus.

---

# 38. Current Special Vocabulary

The language is intentionally small.

## General

```text
let
const
if
else
for
while
loop
await
within
return
```

## Real-Time

```text
realtime
every
on
minimum_interval
deadline
priority
expose
failsafe
fault
limit
```

Some of these may ultimately be ordinary library or declaration concepts rather than hard keywords.

## Reactive

```text
watch
state
transition
```

This is intentionally tiny.

---

# 39. Concepts Deliberately Not Added

At the current stage, the design does **not** add:

```text
event
heal
recover
proof
invariant
mutex
semaphore
thread
lock
unlock
malloc
free
try
catch
```

Some may eventually exist in lower-level runtime libraries or the future proof system, but none is currently needed as a central language abstraction.

The design should resist vocabulary growth unless a concrete problem cannot be expressed cleanly with existing constructs.

---

# 40. Representative Example

The following example demonstrates the current shape of the language.

```text
const MAX_SPEED = 2m/s
const MIN_VOLTAGE = 10.5V


realtime MOTOR_CONTROL every 1ms

    input target_speed

    let speed = encoder.speed
    let voltage = battery.voltage

    let error = target_speed - speed
    let output = pid(error)

    limit output to -75% .. 75%

    motor.left = output
    motor.right = output

    expose speed, voltage

end


state STOPPED
    MOTOR_CONTROL.speed == 0m/s
end


state SAFE
    MOTOR_CONTROL.speed == 0m/s
    BRAKE.engaged
    MOTOR_CONTROL.enabled == false
end


transition MOVING -> STOPPED

    MOTOR_CONTROL.target_speed = 0m/s

    await MOTOR_CONTROL.speed == 0m/s within 5s
    else
        fault STOP_TIMEOUT
    end

end


transition STOPPED -> SAFE

    BRAKE.engaged = true
    MOTOR_CONTROL.enabled = false

end


watch MOTOR_CONTROL.voltage

    if MOTOR_CONTROL.voltage < MIN_VOLTAGE

        let attempt = transition SAFE

        await attempt.status == SUCCEEDED or attempt.status == FAILED

        if attempt.status == FAILED
            transition EMERGENCY_SAFE
        end

    end

end
```

The exact syntax will evolve, but this example captures the semantics currently intended.

---

# 41. Design Principles to Preserve

The following principles have emerged repeatedly and should survive future syntax changes.

### 1. Do more with less

Prefer a few orthogonal concepts over a large feature vocabulary.

### 2. Put theory in the compiler, not in the programmer's head

Dimensional analysis, dependency tracking, scheduling defaults, and state relationships should be handled by the language/toolchain wherever practical.

### 3. Physical meaning and machine representation are separate

`m`, `s`, `V`, and similar units describe meaning.

`i32`, `f32`, `f64`, etc. describe storage and arithmetic representation.

### 4. Real-time code should be deterministic by default

Potentially unbounded or nondeterministic behavior should be explicit.

### 5. Published reality should be coherent

`expose` provides a deliberate synchronization boundary.

### 6. Reactive behavior should be dependency-driven

Watch actual changing values rather than continuously reevaluating arbitrary conditions.

### 7. State describes reality

States are predicates, not ceremonial labels.

### 8. Transition requests are declarative

Ask for a desired state; allow deterministic planning machinery to determine the known legal path.

### 9. Self-healing is behavior, not a special language feature

Use `watch` plus ordinary logic and transitions.

### 10. Intelligence belongs above deterministic machinery

Autonomy may decide what should happen. Reactive and real-time layers retain control over what can safely and deterministically happen.

### 11. Proofing comes last

First stabilize execution semantics. Then build static reasoning over a well-defined model.

### 12. The compiler describes; RV-9 enforces

R9 should communicate requirements and static evidence in machine-readable form. RV-9 should admit against actual resources, enforce what it can, measure what it cannot prove, and make disagreement observable.

---

# 42. Open Questions

The following remain intentionally unresolved.

## Real-Time

Mostly implementation and target-contract details:

- final R9 bundle, publication, and mailbox ABI with RV-9;
- exact RV-9 admission and scheduler contract;
- exact default priority policy;
- final syntax for event-released components;
- whether `budget` or `phase` is eventually necessary;
- final overflow rules for physical quantities;
- exact input declaration syntax;
- details of RT-safe standard library functions;
- machine-readable device latency and ownership metadata;
- exact restricted failsafe representation (see §15.1 for `fault`, now defined);
- how a transition body returns a faulted component to service.

## Reactive

Mostly implementation/planning details:

- exact internal transition-planning algorithm;
- how overlapping predicates/states are represented;
- whether planner cost is ever needed;
- whether transition execution may be parallelized;
- how planner cycles and impossible goals are diagnosed;
- exact lifetime and scoping rules for transition result objects.

The architecture itself is considered settled enough to implement and test.

## Autonomy

Still open:

- whether autonomy needs additional language constructs at all;
- whether Conatus directly supplies the autonomy layer;
- interface between CC/RTC and transition planning;
- representation of goals and drives;
- uncertainty and probabilistic reasoning;
- long-term memory/context representation;
- how novel discovered behavior becomes reusable deterministic machinery;
- whether an explicit autonomy API is sufficient.

---

# 43. Current Working Summary

The language currently has a clear two-layer deterministic foundation.

### Real-Time

```text
realtime
every
on
minimum_interval
deadline
within
expose
```

It executes periodic or event-released physical behavior predictably, publishes coherent observations, and compiles its resource requirements into contracts that RV-9 can admit, enforce, and measure.

### Reactive

```text
watch
state
transition
```

It observes published reality and uses known transformations to move the system toward declaratively defined states.

### Autonomy

Not yet finalized.

The strongest current hypothesis is that **Conatus may be the autonomy system**, while this language supplies the safe deterministic substrate that connects intelligence to the physical world.

That is the next major design question.
