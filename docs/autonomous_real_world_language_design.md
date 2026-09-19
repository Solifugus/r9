# Rachis9 (R9)

## Language Design for Autonomous Real-World Systems

**Status:** Draft design capture  
**Scope:** The architecture of all three layers is now covered. REALTIME is relatively mature. REACTION is well defined in shape, with several constructs still provisional. PROACTION has a settled architecture — serialized authority, authority domains, a bounded log, ordinary program data — and no grammar.  
**Open area:** PROACTION's grammar, and the concrete syntax of the language as a whole, which §40 gives by example rather than by definition. §42 lists what remains open in each layer.

**Markers.** Where the distinction matters, text is marked:

- **Settled**: an architectural decision, changed only deliberately;
- **Provisional**: syntax or semantics used in examples, still subject to refinement;
- **Implementation question**: the meaning is settled, but how R9 or RV-9 realizes it is not;
- **Open research**: the problem is recognized, and no answer is proposed.

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
- provide a natural path from low-level REALTIME control through REACTION to PROACTION, including autonomous behavior on the system itself;
- avoid unnecessary constructs when existing ones can express the same idea.

A recurring design principle is:

> **Do more with less.**

Reasonable behavior should come from the shortest form. Additional syntax should refine behavior, not create basic correctness from scratch.

---

# 2. Architectural Model

The system has three layers:

```text
            PROACTION
    objectives / deliberation
   memory / knowledge / learning
               |
   requests states; observes results
               v
            REACTION
    watch / state / transition
               |
               v
            REALTIME
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

**Settled.** The formal names of the layers are **REALTIME**, **REACTION** and **PROACTION**. They correspond to the RV-9 execution classes of the same names, carried in the manifest's `class` tag. RV-9 does not yet act on that tag, and it schedules REACTION and PROACTION work as ordinary processes. The words *real-time*, *reactive* and *autonomous* are still used descriptively.

### REALTIME

> Perform physical operations correctly and on time, and publish coherent reality.

### REACTION

> Given observed conditions, deterministically respond and reach known states.

### PROACTION

> Decide what should happen next, including acting on its own initiative.

These statements are conceptual, not syntax. REALTIME is relatively mature. REACTION is becoming well defined. PROACTION is under active design (§34).

**Settled:** authority runs downward, from REALTIME constraints and safety to REACTION to PROACTION (§31). PROACTION is part of the system itself, not merely an interface to an outside intelligence, though it may use outside services (§34).

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

**As built, 2026-09-13:** the first of these now exists, in two halves (RV-9 `docs/design.md` §34).

- **`docs/target/rv9-profile.json`** is generated from RV-9's sources and checked for staleness by its tests. It covers:
  - the module format and ABI;
  - every manifest tag, with its number, encoding, repetition, value names, and whether anything enforces it;
  - execution classes;
  - every fault code with its R9 reason;
  - process and I/O error codes;
  - every call in the module environment, with the ABI version that added it and whether it is real-time safe (`yes`, `device`, `no`);
  - which file managers and drivers are real-time safe;
  - real-time and process limits.
- **The `profile` command** prints what one particular board offers: its devices, their file managers and drivers, and its real-time priorities relative to the radio.

Six registered manifest tags have no consumer in RV-9's firmware yet: `desc`, `static`, `class`, `capability`, `compiler` and `runtime`. `desc` is read only by a host-side tool. The profile marks all six unenforced, so a compiler should not treat emitting them as a guarantee.

This closed loop lets the compiler reject known-invalid programs before deployment while allowing RV-9 to recheck claims against the actual machine and report when measured behavior disagrees.

The intended initial deployment model is a compiler-generated bundle:

| R9 concept | RV-9 representation |
| --- | --- |
| R9 program | A related bundle of RV-9 modules and metadata |
| Real-time component | An independently admitted and schedulable real-time program |
| REACTION layer | An ordinary bounded supervisor program |
| Published values and inputs | Fixed-size publication cells and mailboxes supplied by the R9 runtime contract |
| States and transitions | Compiler-generated tables used by the reactive planner |
| Failsafe declaration | Mandatory data interpreted by RV-9 or a trusted supervisor, not cleanup code in the failing process |
| PROACTION | Not yet decided (§34) |

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

Core machine types:

```text
bool

i8    i16   i32   i64
u8    u16   u32   u64

f32   f64

fixed[a,b]

text[n]        // bounded UTF-8 text (§4.1)
instant        // a point in time, not a duration (§5.1)

enum
record

type[n]        // fixed-size arrays, any number of dimensions (§4.2)
```

Vectors, matrices, quaternions, transforms and estimates are **not** core types. They are library types built over these once parametric types exist (§44.3).

**`f128` was removed.** Nothing among the libraries R9 expects (Appendix A) needs it, and `f64` already resolves to about a nanometre over the circumference of the Earth. Long-range navigation error is dominated by sensors, wind and models, and the loop is closed continuously: precision beyond `f64` buys nothing that the next fix does not.

**`fixed[a,b]` stays, for a reason worth recording.** Where a target has no hardware floating point, `f32` is emulated in software, and a 1 kHz control loop is exactly where that cost lands. Whether the first RV-9 targets have an FPU is a question for the RV-9 side (§42); if they all do, this type should be reconsidered.

Large dynamic runtime structures are not required for the REALTIME layer. PROACTION will need them (§34.4); this section describes what REALTIME and REACTION rely on.

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
text[64]
```

There is no `char` type; see §4.1.

## 4.1 Text

**Settled; spelling provisional.**

Text is needed in more places than a control loop suggests: log entries (§34.4), display labels, telemetry, operator messages, and anything exchanged with an outside service.

- **UTF-8 bytes, with a length.** Not NUL-terminated. A length and a byte count let text hold any byte, make length O(1), and match the accounting R9 already does, since a log entry's maximum length is in bytes.
- **Capacity is part of the type.** `text[64]` is 64 bytes of capacity plus a length, so text needs no allocation and can be used in REALTIME, where nothing may allocate.
- **Concatenation is statically bounded.** `text[16] + text[8]` has type `text[24]`, and the compiler checks that a result fits where it is put. Truncation happens only where a program asks for it.
- **Literals are double-quoted**, with `\n`, `\t`, `\\`, `\"` and `\u{...}`. **An unrecognised escape is a compile error**, never a character passed through silently.
- **Offsets are byte offsets.** Iterating by code point is a library function. There is no `char` type: under UTF-8, a fixed-width character type is a trap.

Storage and display are different problems. A target whose console font covers only ASCII will not render text that stores perfectly well (§42).

## 4.2 Arrays, and What They Are Not

**Settled.**

An array is storage with a shape, in any number of dimensions:

```text
f32[3,3]             # a 3x3 of floats
u8[480,640,3]        # an image
f32[1024,3]          # a point cloud
```

Below PROACTION every dimension is a compile-time constant. Arrays that grow belong to PROACTION, and wait on the heap question (§34.4).

**An array is not a matrix.** Defining `*` on a two-dimensional array forces a choice between elementwise multiplication and the matrix product, and either answer silently does the wrong thing to half of those who use it. So arrays offer indexing, slicing and elementwise operations, while `matrix<3,3>`, `vector<3>`, `quaternion` and `transform` are **library types** over array storage, once parametric types exist (§44.3). There `*` is the matrix product, because the type says so.

**Tensor algebra is not planned.** Storage of three and four dimensions is needed — images, voxel grids, inference buffers — but contraction and its relatives belong to neural inference, which R9 delegates rather than computes (Appendix A, tier 4).

**Dimensions stay off the array.** A rotation matrix is dimensionless, a Jacobian's entries are m/rad, and a covariance matrix carries squared units: one dimension for a whole array fits none of them. Units live on the vectors and quantities an array is applied to. This is a real limitation rather than a tidy answer, and it is what makes unit-checked linear algebra awkward in every language that has tried it (§42).

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

## 5.1 Instants, Durations and Angles

**Settled; spelling provisional.**

Two distinctions that dimensional analysis alone does not make.

**An instant is not a duration.** `5ms` is a duration, a quantity of dimension time. A timestamp is a point in time: a publication carries an observation time (§18.1), a log entry carries `time` (§34.4), and a relative view computes one instant minus another. Adding two instants is meaningless, and until now nothing in R9 said so:

```text
instant - instant   -> duration
instant + duration  -> instant
instant + instant   -> an error
```

**An angle is not dimensionless.** In SI a radian is m/m, which is why torque (N·m) and energy (J) share dimensions, and why an angle can be added to a bare number unchallenged. Kinematics, quaternions and IMU fusion all live on angles, so R9 treats **angle as a base dimension of its own** rather than inheriting that ambiguity. `rad` is the unit; `deg` is accepted as a literal form and converted at compile time.

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
await abs(MOTOR_CONTROL.speed) < 0.01m/s within 5s
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

# 11. REALTIME LAYER

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
placement
```

`placement` is optional, and most components should not use it (§13.1).

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

### `placement`

Most programmers should not need to think about scheduling order.

RV-9 derives scheduling priority from the complete admitted workload and its periods, deadlines, minimum intervals, and execution bounds. A faster period alone is not always enough to choose correctly.

An explicit declaration remains as an escape hatch. It is a constraint on RV-9's placement rather than a number (§13.1):

```text
placement urgent
```

Earlier drafts spelled this `priority 10`. `priority` is no longer a word in R9.

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

**The escape hatch is a constraint, not a number.** Decided 2026-09-13 and built (RV-9 `docs/design.md` §36). With two levels, an explicit `priority 10` would have nothing to map onto, so RV-9 offers a declared *placement* instead: manifest tag `placement` (0x0014), one of `derived` (the default), `urgent` or `routine`. A pinned component is never the one moved to make room. If its pin leaves any loop unable to meet its deadline, it is refused `UNSCHEDULABLE` like any other component, so the hatch cannot be used to defeat the analysis. `urgent` also answers the second question below: it is how a component says it must never run beneath the radio. A spelling in R9 might be:

```text
placement urgent     # never below the radio
placement routine    # never ahead of it
```

The value names are published in the target profile under `placement`.

**Open for this document:**

- **The surface syntax** for placement is provisional. Whether `priority` stays as a word is settled: it does not (§13).
- **Routine components run below the radio**, and their bounds do not include it. `placement urgent` keeps a component out of that position but does not bound the radio. Whether a routine bound should ever be trusted for a hard deadline is still a language decision.

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

## 14.1 Properties Travel Through the Call Graph

**Settled in principle; the inference is an implementation question.**

Several restrictions in this document are stated about a construct:

- an admitted REALTIME path may use only operations RV-9 marks bounded and real-time safe (§16.1);
- a `failsafe` may not allocate, wait, or call arbitrary functions (§15);
- a `require` is evaluated while planning, so it may not have effects (§25.1);
- a watch body may not fan out (§21, §25.2).

A restriction on a construct is worthless if a function call can carry that construct past it. Each restriction is therefore a property of a *function*, and the compiler propagates it through the call graph. A function that contains `in parallel`, or reaches one through anything it calls, carries that property, and a context that forbids it rejects the call.

R9 can do this. A program is compiled as a bundle, and the layers where these restrictions apply have no closures, no reflection and no arbitrary indirection, so the call graph is knowable.

- **Inference rather than declaration** keeps the vocabulary small. Nothing is annotated, and the compiler works it out.
- **The diagnostic matters as much as the check.** An error must name the call path that reached the offending construct, not merely the function that was called.
- **Open:** whether some properties should also be declarable, for separate compilation or for a library whose source is absent. Indirect calls are answered in §44.2: permitted in PROACTION, where the compiler takes a property to hold only where it holds for every possible target, and absent below it.

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
    await abs(MOTOR_CONTROL.speed) < 0.01m/s within 5s
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

Step 1 is empty because a transition owns no devices; steps 2 and 3 are §29's existing semantics. Nothing new is introduced — the keyword names an outcome the REACTION layer already had.

The intent the programmer expresses is identical in both places: *I have detected that I cannot continue correctly; do the defined thing.* Only the consequence differs, and the compiler knows the context.

## 15.3 Faults nobody wrote

`fault NAME` is the *programmed* fault. Programs cannot anticipate every failure, so the mechanism above must exist regardless — and everything that stops a real-time component abnormally uses it. The keyword names an instance of a mechanism, rather than introducing one:

| origin | named by | reason |
| --- | --- | --- |
| `fault NAME` | the program | `NAME` |
| deadline missed | RV-9 | `DEADLINE` |
| stopped responding — never returned to wait | RV-9 | `DEADLINE` |
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

**Decided 2026-09-13: a runaway is a `DEADLINE`.** A component that stops coming back to wait has missed every deadline since it stopped, and a program can do nothing different about "stopped responding" than about "late": in both cases the component has stopped, its failsafe has been applied, and its outputs are not to be trusted. The language therefore has no separate word for it. §15.3's table has a row mapping it to `DEADLINE`, and a publication cell reports `DEADLINE`. RV-9 keeps the name `RUNAWAY` in its process table and log, because how a component failed matters to whoever is debugging it, even though it does not matter to the program reacting to it.

The mapping holds only if every `realtime` component has a deadline. A component released by `on` with no `minimum_interval` has none, so "late" is undefined for it. So **every `realtime` component must declare a release bound**: `every`, or `on` together with `minimum_interval`. A compiler should reject one that declares neither. RV-9 already cannot analyse such a component for admission (§13.1), so this closes a gap on both sides.

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

**Declared, since.** This section first noted that a cell was created by whoever first opened it for writing, so two components agreed on a name only by convention, and that declaring cells in the module manifest was the natural next step. RV-9 has since taken that step (§15.6). `publishes` reserves a cell at admission, so a second publisher is refused before it starts. `watches` is refused when nothing on the machine could ever publish the name. A compiler emits `publishes` from `expose` and `watches` from `watch`.

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

**Provisional:** an input may declare the value a component uses until the input is first written, as in `input enabled = false`. A component is then never left acting on an input nobody has set. Before that first write, the input's cell has never been published (§18.1).

Inputs are written by REACTION. PROACTION does not write them, even when it runs on the same machine (§31.2).

---

# 20. REACTION LAYER

The REACTION layer deliberately introduces very little new vocabulary.

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

Transitions take one refining word, `require` (§25.1), and two block forms for ordering work, `in sequence` and `in parallel` (§25.2). `watch` can also handle silence (§21.1).

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

**A watch body is a reflex.** It observes and responds. It is not where work is orchestrated, and it may not use `in parallel`, either directly or through anything it calls (§14.1, §25.2). Where a response genuinely needs concurrency, that belongs in the transition the watch requests, where safe preemption points and failure are already defined (§31.4).

## 21.1 Silence

**Required; syntax provisional.**

`watch` reacts to publication. A physical system must also notice when an expected publication *stops*: a sensor is disconnected, a process stops publishing without faulting, or a link goes quiet. A publication that never arrives changes nothing, so a plain `watch` never runs.

A promising form reuses `within ... else` from `await` (§10):

```text
watch IMU.orientation within 50ms
    process_orientation()
else
    transition SAFE
end
```

The body reacts to publication or change, as before. The `else` path handles a failure to publish within the required interval.

Silence is handled this way without an event subsystem (§22). Absence becomes a branch of the construct that already observes the value.

**Still to be refined:**

- with several watched references, whether the interval applies to each of them or to any publication among them;
- whether `else` runs once per silence, or again for each interval the silence lasts, and what happens when publication resumes;
- when the first interval starts, if nothing has been published yet;
- whether the interval is measured against publication time or against the observation time a cell carries (§18.1);
- how silence relates to a faulted publisher, whose cell already reports the fault at once (§15.6). Silence should not be the only way a known fault is noticed.
- **Implementation question:** for this to cost nothing while a value is healthy, RV-9's blocking wait on a cell's sequence needs a timeout.

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

    abs(MOTOR_CONTROL.speed) < 0.01m/s
    abs(MOTOR_CONTROL.power) < 0.5%
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

## 23.1 Tolerance

**Required; form provisional.**

Measured physical values are never exactly anything. A cart at rest reports speeds a hair either side of zero, so:

```text
MOTOR_CONTROL.speed == 0m/s
```

is almost never true, and a state defined by it is almost never reached.

**Provisional rule:** the compiler rejects exact `==` and `!=` on physical quantities with a continuous representation when they appear in state predicates, `require` conditions and `await` conditions. It warns about them elsewhere. Integers, enums and booleans compare exactly, as usual.

Until a dedicated form is chosen, a tolerance is written as an ordinary bound, and named where it carries meaning:

```text
const STILL = 0.01m/s

state STOPPED
    abs(MOTOR_CONTROL.speed) < STILL
end
```

The compiler checks a tolerance's dimension like any other quantity. A dedicated spelling might prove clearer, whether a `near(value, target, tolerance)` function or a tolerance attached to the comparison itself. That choice is open.

This matters more in R9 than in most languages. Planning, preemption and recovery all reevaluate the actual physical state (§31.4), so a predicate that can never quite be satisfied would break all three.

**Open:**

- **Hysteresis.** A noisy value sitting near a boundary makes a predicate flicker, which fires watches repeatedly and makes a goal alternate between reached and not reached. It is not decided whether states need hysteresis or a dwell time, or whether a program expresses this with two thresholds, as §40 does.
- **Tolerance against the sensor.** A tolerance finer than a sensor's resolution or noise can never be relied on. The compiler could check this only with device metadata it does not yet have (§42).

## 23.2 Parameterized States

**Provisional.**

A state may take parameters, and then describes a family of physical conditions rather than one:

```text
state AT(target: position)
    distance(NAV.position, target) < 0.2m
end
```

One declaration replaces `AT_DOCK1`, `AT_DOCK2` and so on. A request supplies the arguments:

```text
transition AT(DOCK_1)
```

A transition definition names the parameter it is written for:

```text
transition STOPPED -> AT(target)
    ...
end
```

Inside that body, `target` is the requested value. The parameters belong to the declarative target state.

**Settled: the graph stays static.** Parameters are values, not new states. The compiler knows every state declaration and every transition definition, and a request binds arguments without adding anything to the graph. Planning searches the same finite graph whatever the arguments are. Parameterization never implies modifying the transition graph at runtime.

For now, parameters belong to the state being requested:

- a parameterized state appears as the *target* of a transition definition, and its parameters are bound by the request;
- it does not appear as a *source*. A system already at some position is recognized by the unparameterized states it also satisfies. In §40, a cart at one position is also `STOPPED`.

It is open whether parameterized sources or intermediate parameterized states are needed, such as a path through `WAYPOINT(p)`, and how the planner would bind them.

**Arguments are checked.** A parameter's dimension and shape are known at compile time. They may be declared, as with `target: position` above, or inferred from use (§6), as in §40. Which of these is required is provisional.

- An argument of the wrong dimension is rejected before planning: at compile time when the request is R9 source, and on arrival when the request comes from outside.
- Limits on the *value* of an argument are requirements (§25.1), so a request outside them fails with a reason that names the limit.

`position` and `distance` here stand for a vector-of-length type (§5) and a library function. How such types are named is not yet specified.

---

# 24. State Semantics

The design explicitly chooses:

> **State conditions define the state.**

They are not merely preconditions for entering the state.

Thus:

```text
state PARKED
    abs(MOTOR_CONTROL.speed) < 0.01m/s
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

    await abs(MOTOR_CONTROL.speed) < 0.01m/s within 5s
    else
        fault STOP_TIMEOUT
    end

end
```

Another:

```text
transition STOPPED -> PARKED

    BRAKE.engage = true

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

## 25.1 Requirements: `require`

**Settled: requirements decide eligibility. Where they may appear, and their exact semantics, are provisional.**

A transition may state conditions without which it is not allowed at all:

```text
transition STOPPED -> AT(target)

    require BATTERY.charge > 20%

    NAV.goal = target

    await NAV.arrived within 10min
    else
        fault NAV_TIMEOUT
    end

end
```

A transition whose `require` condition is false is not currently an eligible edge for the transition planner. The planner does not try it and give up. For the purpose of that plan, the edge does not exist.

`require` and `if` mean different things, and both are needed:

- `if` is ordinary runtime branching inside a body that has already been chosen;
- `require` decides whether the body can be chosen at all.

A requirement also differs from a source state. `STOPPED` says what the world must be like for the body to make sense. `require` adds conditions that belong to no state, such as permission, resources, or the limits on an argument, without multiplying states to express them.

**Provisional rules:**

- `require` lines come first in a body, before any statement with an effect, because they are evaluated when planning rather than when execution reaches them.
- A requirement may refer to published values, constants and the transition's parameters. It may not refer to a body's local variables.
- A plan evaluates the requirements of all its steps against the values observed when the plan is made. Those values can change while earlier steps run, so each transition's requirements are checked again when it is reached. If one is false then, the attempt fails with `PRECONDITION_FAILED`. It is open whether the attempt should instead replan from observed reality.
- The compiler knows each requirement's dependencies, and emits `watches` for them like any other observation (§15.6).

**Diagnosis.** Because requirements are kept separate from the graph, a failure can say which requirement mattered. Suppose a path exists in the graph, but every such path is blocked by some requirement. The attempt then fails with `PRECONDITION_FAILED` rather than `NO_PATH`, and it carries the blocking requirements (§29).

## 25.2 Ordering Work: `in sequence` and `in parallel`

**Provisional.**

Statements in a transition body run one after another. Many physical transformations are not like that. A karate move, a boxing combination, or a machine that raises an arm while it drives is several things happening at once, which together are one transformation from one state to another. That is a transition, and its body needs to be able to say so.

```text
transition AT(DOCK) -> LOADED

    in parallel

        in sequence
            ARM.target = BIN
            await ARM.at_target within 10s
            else
                fault ARM_TIMEOUT
            end
            GRIPPER.close = true
        end

        in sequence
            CONVEYOR.speed = 0.2m/s
            await CONVEYOR.ready within 8s
            else
                fault CONVEYOR_TIMEOUT
            end
        end

    end

end
```

Blocks nest, so a branch may contain either kind. `in sequence` is what a body already does, and it earns its place mainly by grouping the steps of one branch inside `in parallel`.

**Concurrency is allowed where it is unambiguous.** The branches of an `in parallel` must not interfere:

- no two branches write the same input;
- no branch reads what another branch writes.

The compiler checks this from dependency information it already has (§17, §15.6). Overlapping writes are an error at compile time, not a race to be found later. Where branches are non-interfering, every possible interleaving produces the same result, so this concurrency costs nothing in determinism. Synchronous languages have long relied on the same argument.

**Completion and failure:**

- the block completes when every branch has completed;
- a `fault` in any branch stops the other branches at their next safe preemption point (§31.4) and ends the block with that reason. Otherwise a hazard is easy to build, with one branch failed while another keeps driving the machine;
- worst-case preemption latency for the block is the longest stretch between safe points in any of its branches.

**Branches need not be threads.** A branch runs until it reaches an `await`, which is both where it waits and where it is safe to leave it. A supervisor can therefore interleave every branch of a body within one process, with no locks and no extra stacks. On a board with kilobytes to spare, that matters.

**This is concurrency of intent, not synchronized motion.** `in parallel` says two things are under way at once. It does not make them finish together. Motions that must be coordinated in time, such as two arms arriving at the same instant, belong to a REALTIME component that commands them, or to a state parameterized by time (§23.2).

**Still to be refined:**

- whether a form that completes when the *first* branch completes is needed, or whether `await A or B` covers it;
- whether a branch may request a transition, and whether the rule is then the same non-interference test applied to whole transitions (§31.3);
- what interference means when one branch affects another through a component's physical behavior rather than through an input.

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

A request for a parameterized state supplies its arguments, as in `transition AT(DOCK_1)` (§23.2). Who made a request matters when two requests conflict (§31.3).

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

Failure reasons:

| reason | meaning |
| --- | --- |
| `NO_PATH` | no sequence of transition definitions leads from the observed state to the requested one, whatever their requirements |
| `PRECONDITION_FAILED` | paths exist, but a `require` blocks every one of them; or a requirement was false when its transition was reached (§25.1) |
| `TIMEOUT` | reserved for an attempt that exceeds a bound it was given; a body that names its own fault reports that name instead (§15.2) |
| `COMPONENT_FAILED` | a component the attempt depends on has faulted (§15.5, §15.6) |
| `CONSTRAINT_VIOLATION` | reserved; the constraint mechanism it would report is not yet designed |
| `PREEMPTED` | the attempt was superseded by higher-authority behavior (§31.3) |

A programmed `fault NAME` in a transition body reports `NAME` (§15.2).

These may simply be enums.

**A reason is not always enough.** `PRECONDITION_FAILED` is most useful when it says *which* requirement blocked the way. Likewise, `COMPONENT_FAILED` should say which component failed, and a programmed fault should say which transition raised it. An attempt should carry that detail beside its reason, for example the blocking requirements identified by name or source location. How that detail is represented is not decided.

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

# 31. REACTION Planning, PROACTION, and Authority

Two different questions meet at this boundary.

### REACTION transition planning asks:

> Given a requested state, can I reach it now, using known legal transformations?

### PROACTION asks:

> Which state should be pursued next, and why?

Example:

```text
let attempt = transition AT(CHARGER_A)
```

If it succeeds, REACTION has completed a deterministic task.

If it returns:

```text
FAILED
reason = NO_PATH
```

REACTION has done its job.

PROACTION may then:

- choose another charger;
- change objectives;
- gather information;
- wait;
- ask for assistance;
- create a new plan;
- abandon the original objective.

This keeps deterministic machinery deterministic, and keeps the decision about what to pursue where it belongs.

## 31.1 Why REACTION and PROACTION Are Separate

**Settled.**

Both layers plan, so planning is not what divides them. Three things do, and they are one thing seen from three sides.

**Provability.** REACTION is a closed world: a finite compiled graph, predicates over declared publications, and a search that terminates. Everything §33 hopes to prove rests on that closure. PROACTION is an open world of unbounded knowledge, learning, heuristics, and services outside the machine. It can be observed but not proven. The boundary between the two layers is the boundary of static analysis.

**Authority follows provability.** REACTION can hold authority because its behavior can be established before it runs. PROACTION cannot, so it proposes (§31.2). Merged, there would be no layer through which a learned or remote suggestion has to pass.

**Survivability.** REACTION is bounded in time and memory, and it must keep working while PROACTION is deliberating, waiting on a remote service, out of memory, or dead. That calls for separate processes, separate memory classes and separate failure, which RV-9 provides.

A test for where something belongs:

- Can it be checked statically, bounded, and made to terminate? Then REACTION.
- Does it need knowledge beyond the current publications, or uncertainty, or learning? Then PROACTION.
- Must it still work when everything above it is gone? Then REACTION or REALTIME.

**The sharpest single difference is memory of progress.**

> REACTION knows only what reality shows. PROACTION knows what it has been through.

REACTION does hold a position within a running transition body, but that progress is always discardable: preemption throws it away and plans again from observed reality (§31.4). PROACTION's memory is the opposite. It is authoritative, it is consulted, and it should survive a restart. A sequence of objectives, whose progress is remembered rather than observed, is therefore PROACTION's kind of thing (§34.3).

§37's crystallization needs this seam as well: behavior can move downward only if there are two layers between which to move.

## 31.2 Authority

**Settled.**

```text
REALTIME constraints and safety
    >
REACTION
    >
PROACTION
```

- **REALTIME** holds what nothing above it may override: timing contracts, `limit`, and failsafes applied by RV-9 below a failed component (§15). REACTION reaches a component only through its declared inputs (§19), and the component's own limits still apply to whatever it is sent.
- **REACTION** owns the inputs and the transition graph, including each transition's requirements (§25.1). It decides whether and how a requested state can be reached.
- **PROACTION** decides what should happen and asks for it. It requests states, and observes published values and attempt results. It does not write component inputs or drive actuators directly. This holds when PROACTION runs on the same machine, not only when something outside is involved.

RV-9 can enforce much of this with mechanisms it already has. A cell has exactly one writer (§18.1), so a PROACTION process cannot write an input cell that REACTION owns. Exclusive device ownership (§16.1) means it cannot take an actuator a component holds. Whether PROACTION may own devices of its own, such as a camera, a radio or storage, is open. For sensing, communication and memory it almost certainly must.

## 31.3 Arbitration

**Settled:** the ordering above, and these consequences of it.

- A REACTION request supersedes a running attempt that PROACTION started. The superseded attempt ends `FAILED` with reason `PREEMPTED`, at its next safe preemption point (§31.4).
- A PROACTION request made while a REACTION attempt is running fails with `PREEMPTED`, without being planned.
- A REACTION response that must *continue* to hold is expressed with `require`, not with a lock. Suppose REACTION has reached `SAFE` because the battery is critical. PROACTION may then ask for something else. But while every transition out of `SAFE` requires enough charge, no eligible edge exists, so the request fails with `PRECONDITION_FAILED` and names the requirement (§29). PROACTION cannot override the safety response, and it learns exactly why.

**Provisional:**

- Until whole transitions may run in parallel (§42), one attempt runs at a time. Concurrency *within* a single transition body is a separate matter, and is allowed wherever it is unambiguous (§25.2).
- A request for the state an attempt is already pursuing, with the same arguments and from the same authority, does not supersede that attempt; it refers to it. Without this rule, a `watch` that requests `SAFE` on every publication would keep restarting its own attempt.
- Between two REACTION requests, the later supersedes the earlier, and the earlier is reported `PREEMPTED`. Two things are open: whether REACTION requests need an order among themselves, and whether supersession at equal authority deserves a reason of its own.

## 31.4 Safe Preemption Points

A superseded attempt cannot simply stop wherever it happens to be. A transition body issues commands to the physical world, and some sequences of commands are safe only as a whole:

```text
DRIVE.enabled = true       # the drive takes hold first
BRAKE.engage = false       # and only then does the brake let go
```

Stopping between these two lines leaves the cart held by both. Written in the other order, stopping between them would leave it held by neither.

**Settled:** an attempt is preempted only at a *safe preemption point*. A safe preemption point is a place where the compiler or runtime can guarantee that stopping leaves no unsafe or inconsistent intermediate condition.

Two kinds of point are safe by construction:

- **transition boundaries**, between the steps of a plan, where one transition body has completed;
- **`await`**, where the body has issued its commands and is waiting on the world.

**Provisional:** these are the points that are guaranteed, not necessarily the only ones. The compiler may identify others where it can show that preemption is harmless, such as a stretch of a body that only reads. What that analysis may rely on is open. So is whether consecutive writes to inputs within a body should publish as one coherent set, the way `expose` does (§18).

**Worst-case preemption latency** is the longest execution between consecutive safe points. The compiler should report it for each transition wherever it can bound that execution. An `await` does not lengthen it, however long its `within`, because the `await` is itself a safe point.

**Planning after preemption starts from newly observed reality.** No stored label records that the cart is "halfway to `AT(3.2m)`". The new plan begins from whichever state predicates the published values now satisfy. This follows directly from §23, and it is the strongest argument for §23: a label would be stale at exactly the moment it matters.

Two further consequences follow:

- **Nothing is rolled back.** A preempted body's commands have already reached the world, and the world cannot be undone. The next plan starts from what those commands did.
- **Observed reality may match no declared state.** Then there is nowhere to plan from, and the result is `NO_PATH`. Showing that a program's states cover every reality it can reach is a job for later static analysis (§33). Meanwhile a program can make coverage true by construction, as §40's `MOVING` and `STOPPED` do.

---

# 32. Relationship to Classical AI

The REACTION transition model resembles classical AI state-space planning.

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
abs(MOTOR_CONTROL.speed) < 0.01m/s
BRAKE.engaged
BATTERY.voltage > 11V
```

The compiler can translate those expressions into an internal representation suitable for planning. `require` (§25.1) plays the part of a classical precondition, and a transition's target state plays the part of its effect.

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
- transition topology and requirements;
- transition reachability;
- safe preemption points;
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

```text
every reality the machine can reach satisfies some declared state
```

But no proof syntax should be designed until the execution model is complete.

Decisions made now should keep that analysis possible. That is part of the reason for several of them:

- states are predicates over declared publications;
- the transition graph is fixed at compile time (§23.2, §37);
- requirements name their dependencies;
- preemption happens only at points the compiler can identify.

---

# 34. PROACTION — UNDER ACTIVE DESIGN

PROACTION is the layer that decides what should happen next, including acting on its own initiative. It is the least settled part of R9, and deliberately so. This section records what is decided, what PROACTION must eventually do, and the research directions being explored. It proposes no grammar.

## 34.1 What Is Settled

- **PROACTION is part of R9, and it runs on the system.** R9 is for genuinely autonomous machines. An R9 system must be capable of autonomous operation without depending on any external intelligence service.
- **External services are resources, not the layer.** LLMs, databases, other agents, remote computers and human operators may be available to PROACTION, and it should be able to use them well. None is required. None holds authority that PROACTION itself does not have.
- **PROACTION proposes; the layers below retain authority** (§31.2). It acts on the physical world by requesting states and observing results, even when it runs on board.
- **PROACTION plans above the transition planner, not within it.** REACTION's planner answers whether a known state is reachable. PROACTION decides which states to pursue, in what order and for what reason, and what to do when REACTION says no.
- **PROACTION will be substantially more dynamic than the layers below.** The restrictions of §4 and §14 exist to make REALTIME timing trustworthy. They are not goals for PROACTION.
- **PROACTION encodes no theory of intelligence.** R9 should support simple rule-based autonomy, search and planning systems, learned systems, LLM-assisted agents, Conatus-like architectures (§35), and shapes nobody has thought of yet. The language supplies primitives for building autonomous systems. It does not define what an autonomous mind must be.

## 34.2 What PROACTION Must Eventually Do

**Requirements, not features.** PROACTION must eventually support:

- decision-making and ordinary internal logic;
- choosing objectives, including on its own initiative;
- deliberation, and planning above the deterministic transition planner;
- memory, and the use of accumulated knowledge and history;
- learning or adaptation where appropriate (§37 limits how learning reaches the layers below);
- communication with outside systems (§34.5).

## 34.3 Pursuit: Serialized Authority

**Settled in principle; the mechanisms are provisional.**

Whatever else PROACTION becomes, it has **one serialized authority**: a single point that commits decisions, changes authoritative PROACTION state, and asks REACTION for states. The computation that supports those decisions need not be serial.

> **Serialize authority, not intelligence.**

Vision analysis, route search, memory retrieval, consulting a remote model, predicting what the battery will do: any of these may run concurrently, on another core, an accelerator, or another machine. Their results return to the serialized authority, which decides what to accept, what to change, and what to ask for.

A single serial process is a reasonable first implementation, and on RV-9 it is the obvious one. It is not the architecture.

What serializing authority buys:

- **One writer.** As with a publication cell (§18.1), a single writer over authoritative PROACTION state gives coherence without locking.
- **Order is decided, not raced.** Contention is settled by the order in which the authority commits, not by which computation happened to finish first.
- **Deliberation stays replayable.** What has to be recorded is the order in which results were accepted and decisions committed, not the internal progress of every computation. Concurrency costs nothing in reproducibility. An unrecorded commit order would cost all of it.
- **Restart is defined.** If PROACTION stops, REACTION keeps the machine safe, and what was being pursued is in the store.

### Computation has no authority

What makes this concurrency safe is the rule that already governs the layers, applied within one: **what computes does not commit.**

- A computation returns a result. It does not change authoritative state, and it does not request transitions.
- A result may arrive too late to matter, so the authority must be free to discard it, and anything started must be abandonable.
- A computation that fails, times out or is cancelled produces a result saying so. It does not fault the authority.

**Open:** how concurrent computation is expressed at all — whether `in parallel` branches carry it, whether it is requested and taken up like an attempt, and how a computation says what it needs. None of that is designed. On RV-9 the obvious shape needs no new mechanism: a separate process, with its own memory budget, publishing into a cell the authority watches.

### `in parallel` here is concurrent pursuit

PROACTION uses the same two block forms as REACTION (§25.2), with the meaning this layer gives them. A branch is a pursuit: something being sought, with its own steps and its own waiting.

Whether branches interleave within one process or genuinely run at once is a property of the target, not of the language. What is fixed is that their decisions serialize.

What differs from REACTION is that pursuits may compete for the same part of the machine.

### Authority domains

A complex machine is not one indivisible resource. Driving the base, moving an arm, aiming a camera and talking to an operator are different authorities, and independent pursuits may legitimately direct different ones at the same time.

The likely abstraction is an **authority domain** — navigation, manipulation, vision, communication, power, and so on — where:

- two pursuits needing the same domain contend;
- pursuits over independent domains proceed at once.

This is §25.2's rule one level up: concurrency is allowed where it is unambiguous. In REACTION the compiler proves non-interference over inputs. In PROACTION, domains name that same interference coarsely enough to arbitrate pursuits before their plans are known in detail.

**Open, and deliberately unsettled:**

- how a domain is declared, and how it relates to the components and inputs REACTION owns. A domain is presumably a set of those;
- when a pursuit's domains are known. Which transitions a plan will use is known once it is planned, and replanning may change them;
- how domains are acquired. If a pursuit can hold one domain while waiting for another, deadlock becomes possible. Taking a whole set at once or not at all, in the manner of RV-9's admission, is the obvious way to keep that impossible;
- whether REACTION ever needs to take a domain a pursuit holds, given that it already outranks every pursuit (§31.2).

### Contention is a queue, and losing is a result

Where two pursuits do contend:

- first come, first served, by the order in which the authority reaches each request;
- the winner keeps what it holds until its attempt ends;
- a pursuit willing to wait takes its place in line, and `within` says for how long;
- losing, or waiting too long, produces an ordinary attempt result (§29), never an error. Nothing unwinds. Deciding what to do about a result is what PROACTION is for.

**Open:** starvation is possible, because a pursuit that always arrives second always loses. Safety does not depend on it, since REACTION outranks every pursuit. Ordering pursuits by importance — value, urgency, drive, whatever it comes to be called — is a PROACTION matter, and is not designed.

### Decision points

Each layer has a recurring boundary. REALTIME has its period, REACTION has `await` and transition boundaries, and PROACTION has **decision points**, where the authority takes up what has arrived, results returning from concurrent computation included, and may change course.

A pursuit may not run arbitrarily far without reaching one, so responsiveness has a bound even though deliberation does not. Redirection then has a clean meaning: something can make a pursuit stale, and the process reconsiders at its next decision point rather than being interrupted mid-thought.

**The authority must not block.** A remote model, a database query, a long retrieval or a transition attempt is issued, and then taken up as an event at a decision point. Serialized and blocking would make the machine hostage to the slowest thing it asked for.

### Watchers that inform

A `watch` in REACTION acts. It can request a transition immediately, because it is a bounded, provable reflex (§21).

A watcher belonging to PROACTION does not take control. It observes, records what it noticed, and may mark that as worth attention. The authority decides what to do about it at its next decision point. This is how serialized decision-making survives things happening at unpredictable times.

### Idleness is a condition to watch

A system pursuing nothing is safe, because REACTION does not depend on PROACTION. But it is not autonomous. "Nothing is being pursued" should therefore be an observable condition with a declared fallback: reassess, return to a dock, recharge. This is §30's shape applied to intent — make the absence observable, and respond to it. The form is open.

### Several subsystems, and several machines

Only the committing of decisions is single. An arm and a base may be pursued at once where they are different authority domains, and the computation behind them may run wherever there is capacity for it. Several machines are several PROACTIONs, each with its own serialized authority, cooperating through the interfaces of §34.5.

## 34.4 Data: Research Directions

**Open research.** Nothing in this subsection is a settled language feature or syntax.

An intelligent agent works over far more knowledge than a control loop does, and keeps it for longer than a power cycle. PROACTION will probably need ordinary rich programming facilities plus substantial data manipulation, memory, persistence, retrieval and reasoning. The current exploratory requirements are:

- rich records and nested structures;
- dynamically sized lists;
- dictionaries or maps;
- sets and similar collections;
- graph-like relationships, with traversal and search;
- persistent object identity;
- long-term data storage;
- collections larger than available RAM;
- transparent or low-burden caching between RAM and persistent storage such as flash, SD card or disk;
- efficient query and index facilities, so that persistent collections do not require complete scans;
- streaming and iteration over data larger than memory;
- historical data;
- potentially extensible indexing, such as semantic, vector or other intelligent retrieval.

**These are things some architectures will want, not a list to be implemented.** Only ordinary program data and a historical log are proposed below. Lists, maps, sets, graphs, vector stores and specialised knowledge structures must earn their place through a demonstrated R9 requirement, rather than because agent frameworks commonly have them (§34.1: PROACTION encodes no theory of intelligence). The evidence so far runs that way: an architecture of simple numeric structures and an LLM-assisted one want quite different things, and records, arrays and a log of text serve both. Nothing beyond them has yet been shown necessary.

### Ordinary program data

**Settled in direction; syntax open.**

PROACTION is a general-purpose programming environment, not a log with a program attached. Beyond R9's scalars and physical quantities (§4, §5) it needs at least:

- **records with named attributes**, so related values are grouped and reached by name;
- **numerically indexed arrays**, fixed in size for now.

```text
observation.temperature
observation.time
observation.source

observations[0]
observations[1]
```

Declaration and type syntax remain to be designed.

**Arrays stay fixed in size for the moment.** Dynamic sizing is the first thing in R9 that would need a heap, and RV-9 modules cannot allocate at all today (see *What RV-9 offers today*, below). Whether PROACTION gets one, and how it is bounded per class, is a question for RV-9 rather than something R9 should assume. In the meantime the growth that PROACTION actually cannot avoid — its history — is carried by the log below, which is bounded by design rather than by hope.

### A historical log

**Settled in direction; the form is deliberately minimal.**

A machine that decides what to do next needs to remember what happened. The initial facility is as small as it can be:

> **A log is timestamped lines of text.**

An entry is **bounded text with attributes**. The text is what a program writes. The attributes are what is known about it, beginning with exactly one:

- **`time`** — when the entry was made, assigned by the store.

Later revisions will add more. The text is content, and the attributes are not part of it. An attribute the store assigns cannot be forged by writing a line, which is the division §34.5 relies on. An attribute a *writer* supplies is a claim rather than knowledge, and the two must not read alike.

**Every attribute carries its origin.** Two kinds exist, and the distinction is §14's, applied to knowledge rather than to execution bounds:

- **assigned** — the store knows it, as it knows `time`;
- **declared** — a writer claims it, as with an anchored value (below).

A reader can always ask which it is, and a projection can show it. A program reading attributes as fields receives the origin along with them, so code cannot silently treat a claim as knowledge. Anything a later revision computes for itself, such as an extraction or an index, would be a third kind — **derived** — and would need its own name for the same reason.

*Provisional:* a projection shows values without origin marks unless asked, on the grounds that a view should stay readable. Whether declared values should instead be marked by default is worth revisiting once there is an attribute other than `time`.

Entries can be viewed and searched with absolute times, or as durations relative to a chosen entry or instant:

```text
-5.904s  Battery voltage dropped
-3.212s  Motor current increased
 0.000s  Traction lost
+0.287s  REACTION entered SAFE
```

Relative views are how temporal patterns around a significant event become visible, which is exactly what deciding and learning need. The reference may be an instant or an entry, including one found by a search, so "everything within five seconds of the first traction loss" is expressible.

**Reading is a projection.** A read chooses which attributes to show and how to show them. The entries are the same; the view differs. This is ordinary in tools and unusual in a language — `git log --format` and `ps -o` do exactly this — and it is worth the strangeness here, because a log is read far more often than it is written, and by very different readers.

Two properties keep it honest:

- **Whatever a projection shows, text search can match.** Attributes rendered into a view are text like any other, so `grep`-style searching reaches them, and metadata needs no second, structured query language of its own.
- **Attributes are also reachable as fields.** An entry read by a program is an ordinary record — `entry.text`, `entry.time` — so nothing has to parse back what it just rendered.

**Anchored attributes.** *Provisional.* A value often belongs in the text and in an attribute at once. "wheel slip 0.42" should read as a sentence and be reachable as a number:

```text
traction lost, wheel slip 0.42, motor current 6.1A
                         ^^^^                ^^^^
```

An attribute may therefore be **anchored** to a span of the entry's text. This is stand-off annotation, as corpora, source maps and editor text properties have long done it, and it lets each half do what it is good at:

- **The text stays canonical.** The entry is stored exactly as written, so text search sees the line it expects, and the length bound stays exact.
- **The attribute is a typed view of that span.** The store validates at write time that the span parses as the declared type, so the text and the attribute cannot disagree. For `6.1A` that includes the unit, which brings dimensional analysis (§5) to the text boundary: the attribute is a current, and adding it to a voltage is still an error.
- **A projection may re-render a span from its attribute** — `0.42` shown as `42%`, a current at a different precision — without the stored text changing. So an entry can be read as text, as values, or as text with its values reformatted.

Initially **spans may not overlap.** Nesting and overlap are where stand-off annotation becomes complicated, and nothing here needs them yet.

**Bounded, and curtailed automatically.** Memory is not eternal. A log that grows forever is a fault waiting to happen, so a log is a **FIFO with a declared limit**: when it is full, the oldest entries go. Losing them is ordinary operation, never an error.

**The limit is a number of entries, with a declared maximum entry length.** Entries are what a program thinks in — the last ten thousand things that happened. A count alone bounds nothing, though, since ten thousand lines may be 400 KB or 40 MB. Declaring the longest an entry may be makes the count a real bound:

```text
entries x (maximum entry length + attributes + a small header) = the storage this log can ever need
```

That number is known before the program runs, which puts a log where every other R9 resource already is: declared by the compiler, and admitted by RV-9 against what the machine actually has (§16.1). A log too large for the board is refused at admission rather than discovered at three in the morning. It is also how publication already works — a cell is a fixed-size piece of memory, not an open-ended one (§18.1).

- **A window in RAM over a larger log in storage.** What is held in memory is a bounded window. The log itself may be far larger, on flash, an SD card or a disk. Moving the window is reading, and it costs what storage costs: searching within the window is cheap, and searching beyond it is not. That difference should be visible rather than hidden, as with any other query (*Three concerns, kept separate*, below). The window is sized in entries as well, and so in bytes.
- **Curtailment is exact, and recorded.** Dropping the oldest entry frees exactly one slot, so there is no compaction and no surprise that discarding one long line recovered as much room as fifty short ones. Where entries have been discarded, the log says so. A reader must be able to tell "nothing happened then" from "that is no longer here" — a system reasoning from its own history will otherwise conclude the first when the second is true.
- **An over-length entry is truncated, with a marker, and never rejected.** A log must not fail a write. Losing the tail of one line is better than losing the fact that it happened. An attribute anchored beyond the cut has nothing left to point at, so it is dropped, and the marker makes clear that something was.
- **Large content lives outside the log, named by it.** Anything too big for an entry — a long reasoning trace, an image — is written elsewhere and mentioned by path. The log stays skimmable, which is what makes text searching work, and this needs no new mechanism, because a path is just text.
- **Appending stays cheap.** Space is reclaimed in whole segments rather than line by line, which is also what flash erase blocks want.

**One log.** A system has one history. Nothing in the design needs more yet, and several logs would immediately raise how to search across them and how their limits relate. This is revisitable rather than a constraint; the natural pressure later is one log per subsystem.

**Not in the initial form:** no graph edges, embeddings, tags, similarity or other speculative metadata. Text and a time.

**Attributes are the extension mechanism.** A later revision adds attributes without altering entry text and without invalidating logs already written. Two things follow from that. An attribute may be **absent** on older entries, and a projection must show absence as absence rather than as an empty value that reads like one. And entries need stable identity for anything later to attach to, which argues for identity now, even though nothing yet uses it.

**Searching should eventually be powerful and composable**, in the spirit of `grep` and `sed`: text matching, filtering, temporal windows and transformations. Which functions, and in what syntax, is open.

**Open:**

- **What happens on the way out.** Whether anything is summarised as it is discarded, or entries simply vanish.
- **How a projection is written.** Whether it is a list of attributes, a format, or a query, and how the reference point of a relative view is named.
- **What search matches.** The rendered projection, or the stored text together with predicates over attributes, or both. Anchored attributes sharpen the question: the stored text may hold `0.42` where the projection shows `42%`. Matching what the reader is looking at is the likelier answer.
- **Showing origin.** Whether a projection should mark declared values by default, and what a third, derived origin would be called.
- **Which time.** When something was observed and when it was logged are different, and publications already distinguish them (§18.1). Entries about the physical world probably want the observation time.
- **Cost.** Writing a line must be cheap enough to do often, and searching must be able to use an index rather than reading everything back.
- **Power failure.** What happens to the unflushed tail, and how a partially written entry is recognised when the log is next opened.

### Three concerns, kept separate

One promising principle is to separate:

1. **logical data structure**: what the data is;
2. **physical storage policy**: where it lives, and what is kept in RAM;
3. **query and index strategy**: how it is found.

Conceptually, and not as decided syntax:

```text
persistent list<Observation>
persistent map<ObjectId, Object>
persistent graph<Memory, Relation>
```

A persistent collection might be logically much larger than RAM, while the runtime keeps only an appropriate working set cached in memory. The programmer should ideally work with the logical collection, rather than opening, reading, seeking and deserializing by hand.

Similarly, queries should ideally execute close to the storage representation, so that something conceptually like:

```text
observations.where(...)
```

can use an index or a storage-aware query plan instead of loading every observation into RAM.

The current north star is:

> **Make large, persistent, searchable knowledge feel like ordinary data.**

This is the move §27 already makes for reaching states, applied to memory: state what is wanted, and let the system determine how to obtain it.

### Graphs

Graph-like storage and search deserve particular attention. An intelligent agent may need relationships among observations, experiences, concepts, states, actions, outcomes, evidence, objects, locations and other knowledge.

We do not yet know whether `graph` should be a first-class type, a way of organizing collections, a standard library facility, or something more general.

### Questions deliberately kept open

- the status of `graph`, as above;
- persistent identity across restarts, and across a record type that changes while stored data still has the old shape;
- durability: what a persistent collection guarantees when power fails during a write, and whether that needs transactions;
- how the cost of a query is bounded, and whether the compiler can report a query's plan the way it reports timing evidence;
- how a working set is sized against a process's memory budget, and what is evicted;
- which indexing techniques are practical on small targets, and how new ones are added;
- how recorded history relates to publication. A publication already carries a sequence number and an observation time (§18.1), and these are natural keys for a history of what was observed.

### What RV-9 offers today

- **No allocation in modules.** An RV-9 module gets a stack and a fixed statics area, and nothing else. RV-9's alignment notes treat this as a good starting point: a heap for PROACTION should be *introduced* already bounded and per class, rather than fenced off later from a general allocator.
- **Memory classes and budgets** (RV-9 `docs/design.md` §35). When ordinary work exhausts memory, the result is a refusal, not a failure of control. That is what lets PROACTION be dynamic without endangering REALTIME.
- **Storage.** There is a flash filesystem (`/f0`) and a RAM disk (`/r0`), and an SD card (`/sd0`) is planned. Any persistent collection kept on flash must respect flash wear.

## 34.5 Interfaces

### PROACTION to REACTION

**Settled in principle:** PROACTION normally requests desired states, with arguments (§23.2), and observes published state and attempt results. It does not bypass REACTION to manipulate physical devices.

**Provisional:** the compiler emits a machine-readable description of this boundary, much as RV-9 publishes its target profile. It would contain:

- the published values that may be observed, with dimension and representation;
- the states that may be requested, with parameter dimensions;
- the attempt reasons (§29);
- a hash of the whole, so that a PROACTION built against a different program is refused rather than misunderstood.

The same description would serve PROACTION on the machine and any outside system that reaches it.

**Open:** whether PROACTION may request every state, or only the states a program marks as requestable.

### PROACTION to outside systems

**Open.** R9 should eventually have clean mechanisms through which PROACTION communicates with remote systems, LLMs, databases, other agents, human operators and network services. None is designed yet.

**Settled in principle:** anything arriving from outside is information for PROACTION to weigh. It gains no authority by arriving. An outside suggestion reaches the world only as a request that PROACTION chooses to make, through the boundary above.

**Provisional:** losing a link is published state, like any other failure (§22). Whatever terminates the link can turn absence into a value, for example by publishing that a peer's lease has lapsed, and REACTION responds with `watch`.

## 34.6 Targets, and the Link

**R9 is meant to span autonomous-system targets.** RV-9 on the ESP32-C5 is an important first platform and proving ground. It is not the size PROACTION must fit.

On that board as configured today, RV-9 measures (RV-9 `docs/design.md` §35):

- about 38 KB of heap free at idle, of which about 17 KB is available to programs;
- about 7 KB for each SSH session, leaving 3.4 KB for programs while one command runs inside a session;
- a default memory budget of 32 KB for each process tree;
- no dynamic allocation in modules at all.

PROACTION as §34.2 and §34.4 describe it will not fit there as the board is presently configured. That is a fact about the first target, not a limit on the design. Plausible arrangements include a larger RV-9 target, or REALTIME and REACTION on the C5 with PROACTION on a companion system. Whichever is used, the authority boundary is the same.

When any part of PROACTION, or anything it talks to, is off the board, the link is a **joint R9/RV-9 design problem**. No protocol is chosen. What is known:

- `rshd` has no authentication, so it is not acceptable for a link that can request states.
- SSH is authenticated, but RV-9 supports one session at a time, and a session costs about 7 KB.
- A lighter authenticated transport may fit better. Nothing has been selected.

---

# 35. Conatus: An Example, Not the Target

Conatus is a separately developed architecture for autonomous agents. It is a useful source of analogies for PROACTION. It is **not** the target architecture for R9. R9 should make it possible to implement or integrate a Conatus-like system, but it must not be designed around Conatus.

Earlier drafts of this document described Conatus by analogy. As of its repository on 2026-09-13:

- **It is driven by values, not by drives or goals.** Valence is a number from −1 to 1. *Encoded* valence comes from configuration and is authoritative. *Learned* valence is adjusted by the agent. The current specification does not use the terms "impetitive" or "aversive". It says explicitly that values take priority over goals and instructions.
- **Its Contemplative Cursor (CC) is a large language model.** It is currently a hosted model, and a local model of about 30B parameters on a 24 GB GPU is planned. The CC writes, scores and repairs sequences.
- **Its Real-Time Cursor (RTC) steps through prepared sequences.** Despite the name, it is **unrelated to R9 REALTIME**. It has no periods or deadlines, and its waits last from hours to months.
- **Its sequence corpus is its procedural memory.** The corpus is compressed by lifting recurring structure into named sequences.
- **"Mind Splicer" is not part of the current specification.** The nearest idea, substitution problem-solving, is planned but not built.
- **Its structural gate checks that sequences are well formed.** It is not a physical safety authority, and R9 must not treat it as one.
- **Its deliberation does not run on the current RV-9 target.** It needs a GPU-class machine.

Ideas worth borrowing for PROACTION:

- **Deliberation keeps a runway ahead of execution.** A slow deliberative process prepares work so that a deterministic executor never has to wait for it. PROACTION might prepare requests for REACTION in the same way.
- **Likely success is estimated from history, keyed by context.** R9 attempt results (§29), recorded together with the states that held when each attempt began, are exactly such a history.
- **Degradation is graceful.** A Conatus deployment keeps executing prepared work when deliberation is unavailable. An R9 system should likewise keep its REALTIME and REACTION behavior, and whatever PROACTION can do locally, when a remote resource disappears.

If Conatus is used with R9, it would either be an external resource that PROACTION consults, or a source of ideas implemented within PROACTION. Either way, its choices reach the world through the boundary in §31. Its preference for values over instructions would operate within PROACTION, never above REACTION.

---

# 36. Potential Long-Term Architecture

```text
     OPTIONAL EXTERNAL RESOURCES
   LLMs / databases / other agents /
   human operators / remote systems
                 :
                 :  information, not authority
                 v
             PROACTION
    objectives / deliberation
    memory / knowledge / learning
                 |
    requests states; observes results
                 v
             REACTION
     watch / state / transition
                 |
                 v
             REALTIME
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

This keeps a useful principle:

> **Intelligence proposes; deterministic machinery retains authority.**

PROACTION may request:

```text
transition AT(LOADING_DOCK)
```

REACTION decides whether and how that state can be reached.

REALTIME ensures that the physical actions occur with deterministic timing and safety.

---

# 37. Behavioral Crystallization

**Exploratory, with one settled limit.**

Behavior may begin high in the system and move downward as it becomes understood:

```text
PROACTION discovers a behavior
      ↓
it is exercised and validated
      ↓
it becomes stable and well understood
      ↓
it moves into deterministic REACTION machinery
```

This may allow autonomous systems to become less computationally expensive and more predictable over time.

**Settled:** learning never modifies the compiled REACTION transition graph at runtime. A stabilized behavior may instead be exported or generated as candidate R9 source. It then passes through the same path as anything a person writes:

```text
learned behavior
    -> candidate R9 transition/source
    -> static analysis
    -> proof/checking
    -> compile
    -> deployment
```

What PROACTION learns at runtime may still change what it chooses to request, because that is within its own authority. It cannot change what REACTION will allow.

This preserves the possibility of strong static guarantees.

**Open:**

- the exact workflow, including whether and where human review is required;
- what evidence a candidate must carry;
- how a new REACTION program replaces a running one without leaving the world undefined. RV-9 can already load modules at runtime without reflashing.

---

# 38. Current Special Vocabulary

The language is intentionally small. `placement` has replaced `priority`, which is no longer a word in R9 (§13.1).

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

## REALTIME

```text
realtime
every
on
minimum_interval
deadline
placement
input
expose
limit
failsafe
fault
on stop
```

`on stop` reuses `on`. `fault` is shared with REACTION, where it ends a transition (§15.2). The surface syntax of `placement` and `input` is provisional. Some of these words may ultimately be ordinary library or declaration concepts rather than hard keywords.

## REACTION

```text
watch
state
transition
require
in sequence
in parallel
```

`watch` also takes `within ... else` for silence (§21.1, provisional). States may take parameters (§23.2, provisional).

## Functions and modules

```text
module
use
function
export
returns
```

Provisional (§44.1): functions grouped in modules, private until exported, `x.f(y)` meaning `f(x, y)`, and operators for the symmetric cases.

## PROACTION

Only the two block forms, with the meaning §34.3 gives them:

```text
in sequence
in parallel
```

Nothing else yet, deliberately (§34).

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
priority
```

Some may eventually exist in lower-level runtime libraries or the future proof system, but none is currently needed as a central language abstraction. `priority` was replaced by `placement` (§13.1). Silence is handled by `watch ... within`, not by an event (§21.1).

PROACTION's needs (§34.4) will test this list hardest. Additions made for it should still meet the rule below.

`in sequence` and `in parallel` (§25.2) are not threads. They are scoped, joined where they are written, and checked for non-interference, which is why `thread`, `lock` and `mutex` remain absent.

The design should resist vocabulary growth unless a concrete problem cannot be expressed cleanly with existing constructs.

---

# 40. Representative Example

The following example shows the current shape of REALTIME and REACTION. PROACTION has no syntax yet.

A shuttle cart runs on a straight track, driven by a motor and held by a brake.

```text
# Names this example takes from outside itself:
#   devices    encoder, motor, brake, cell_monitor
#              (bound to RV-9 paths by the build; the binding syntax is open, §42)
#   functions  pid, abs (standard library)

const STILL       = 0.01m/s     # a measured speed below this is "not moving" (§23.1)
const ARRIVED     = 5mm         # how close counts as "at" a position
const TRACK_START = 0m
const TRACK_END   = 6m
const MIN_CHARGE  = 20%         # needed to start a trip
const CRITICAL    = 10%         # below this, stop; the gap up to MIN_CHARGE is hysteresis


realtime DRIVE every 1ms
    deadline 800us

    input goal = 0m             # provisional: the value used until first written (§19)
    input enabled = false

    let position = encoder.position
    let speed = encoder.speed

    let output = pid(goal - position)
    if enabled == false
        output = 0%
    end

    limit output to -75% .. 75%

    motor.power = output

    let powered = enabled
    expose position, speed, powered

end

failsafe DRIVE
    motor.power = 0%
end


realtime BRAKE every 10ms

    input engage = true

    brake.command = engage
    let engaged = brake.closed

    expose engaged

end

failsafe BRAKE
    brake.command = true
end


realtime BATTERY every 100ms

    let voltage = cell_monitor.voltage
    let charge = cell_monitor.charge

    expose voltage, charge

end


# MOVING and STOPPED between them cover every reality,
# so a plan always has somewhere to start (§31.4).

state MOVING
    abs(DRIVE.speed) >= STILL
end

state STOPPED
    abs(DRIVE.speed) < STILL
end

state SAFE
    abs(DRIVE.speed) < STILL
    BRAKE.engaged
    DRIVE.powered == false
end

state AT(target)                # target's dimension is inferred from DRIVE.position (§23.2)
    abs(DRIVE.position - target) < ARRIVED
    abs(DRIVE.speed) < STILL
    BRAKE.engaged
end


transition MOVING -> STOPPED

    DRIVE.goal = DRIVE.position       # hold where it is now
    DRIVE.enabled = true

    await abs(DRIVE.speed) < STILL within 5s
    else
        fault STOP_TIMEOUT
    end

end


transition STOPPED -> SAFE

    BRAKE.engage = true               # the brake takes hold first
    await BRAKE.engaged within 500ms
    else
        fault BRAKE_TIMEOUT
    end

    DRIVE.enabled = false             # and only then is the drive released
    await DRIVE.powered == false within 50ms
    else
        fault DRIVE_TIMEOUT
    end

end


transition SAFE -> AT(target)

    require BATTERY.charge > MIN_CHARGE
    require target >= TRACK_START and target <= TRACK_END

    DRIVE.goal = DRIVE.position       # hold here
    DRIVE.enabled = true              # the drive takes hold first
    await DRIVE.powered within 50ms
    else
        fault DRIVE_TIMEOUT
    end

    BRAKE.engage = false              # and only then does the brake let go
    DRIVE.goal = target

    await abs(DRIVE.position - target) < ARRIVED and abs(DRIVE.speed) < STILL within 2min
    else
        fault TRAVEL_TIMEOUT
    end

    BRAKE.engage = true
    await BRAKE.engaged within 500ms
    else
        fault BRAKE_TIMEOUT
    end

end


watch BATTERY.charge within 1s

    if BATTERY.charge < CRITICAL
        transition SAFE
    end

else
    transition SAFE                   # the battery monitor has gone silent
end
```

What the example shows:

- **Planning from wherever the cart is.** From `SAFE`, a request for `AT(3.2m)` plans the single step `SAFE -> AT(3.2m)`. From a moving cart, the plan is `MOVING -> STOPPED -> SAFE -> AT(3.2m)`.
- **No `AT -> …` transition is needed.** A cart already at one position that is asked for another also satisfies `STOPPED`, so its plan is `STOPPED -> SAFE -> AT(…)`. This falls out of states being predicates.
- **Preemption.** If charge falls below `CRITICAL` in the middle of a trip, the `watch` requests `SAFE`. REACTION outranks whatever asked for the trip, so that attempt ends `PREEMPTED` at its next safe point, the travel `await`. Planning then starts from what is observed, `MOVING`, rather than from where the trip was.
- **Preconditions with reasons.** Until charge is back above `MIN_CHARGE`, a new request for any `AT(…)` fails `PRECONDITION_FAILED` and names `BATTERY.charge > MIN_CHARGE`. A request for `AT(9m)` fails at any charge and names the track limit.
- **Silence.** If `BATTERY` stops publishing for a second, the `else` path requests `SAFE`.
- **Component faults.** If `DRIVE` or `BRAKE` faults, RV-9 applies its failsafe, and the fault is published in the component's cell (§15). A transition awaiting that component fails `COMPONENT_FAILED`.
- **Order of commands.** Every hand-over between brake and drive is ordered so that something always holds the cart, and each hand-over is separated by an `await`. That makes every point where preemption can happen safe (§31.4).

Whatever PROACTION becomes, it would reach this machine only by requesting states such as `AT(3.2m)` and observing the published values and attempt results.

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

Watch actual changing values, and their silence, rather than continuously reevaluating arbitrary conditions.

### 7. State describes reality

States are predicates, not ceremonial labels.

### 8. Transition requests are declarative

Ask for a desired state; allow deterministic planning machinery to determine the known legal path.

### 9. Self-healing is behavior, not a special language feature

Use `watch` plus ordinary logic and transitions.

### 10. Authority stays with deterministic machinery

PROACTION may decide what should happen, including on its own initiative. REACTION and REALTIME retain control over what can safely and deterministically happen.

### 11. Proofing comes last

First stabilize execution semantics. Then build static reasoning over a well-defined model. Meanwhile, make decisions that keep that reasoning possible.

### 12. The compiler describes; RV-9 enforces

R9 should communicate requirements and static evidence in machine-readable form. RV-9 should admit against actual resources, enforce what it can, measure what it cannot prove, and make disagreement observable.

### 13. Reality is re-observed, not remembered

Planning, preemption and recovery start from what published values show now, never from a stored label for where the system believes it is.

### 14. Autonomy is intrinsic

An R9 system must be able to act autonomously by itself. External intelligence is a resource it may use, not a dependency.

### 15. Serialize authority, not intelligence

Work may proceed concurrently wherever it is unambiguous, and computation wherever there is capacity for it. What is serialized is the committing of decisions and the changing of authoritative state (§34.3).

---

# 42. Open Questions

The following remain intentionally unresolved.

These are settled since the first draft and no longer open:

- derived priority and `placement` (§13.1);
- deadline and runaway faults (§15.6);
- the publication ABI and declared cells (§18.1).

## REALTIME

Mostly implementation and target-contract details:

- final R9 bundle and mailbox ABI with RV-9;
- the surface syntax of `placement`, and whether a `routine` bound should ever be trusted for a hard deadline (§13.1);
- final syntax for event-released components;
- whether `budget` or `phase` is eventually necessary;
- final overflow rules for physical quantities;
- exact `input` syntax, including an input's value before its first write (§19);
- details of RT-safe standard library functions;
- how device names bind to RV-9 paths, and machine-readable device latency and ownership metadata;
- exact restricted failsafe representation (see §15.1 for `fault`);
- how a transition body returns a faulted component to service (§15.5).

## REACTION

The architecture is considered settled enough to implement and test.

Provisional syntax and semantics that still need refinement:

- the form of tolerances in predicates, and whether states need hysteresis or dwell (§23.1);
- parameterized states: declared or inferred parameter types, and whether parameterized states may be transition sources or intermediate steps (§23.2);
- `require`: where it may appear, and whether a requirement found false during execution fails the attempt or replans (§25.1);
- the exact semantics of `watch ... within ... else` (§21.1);
- arbitration among REACTION requests, and whether supersession at equal authority needs its own reason (§31.3);
- safe preemption points beyond transition boundaries and `await`, and whether consecutive input writes publish as one set (§31.4);
- the representation of attempt diagnostics (§29);
- `in parallel`: whether a first-to-complete form is needed, whether a branch may request a transition, and what interference through a component rather than an input means (§25.2);
- how far property inference through the call graph should reach, and whether properties should also be declarable (§14.1).

Implementation questions:

- exact internal transition-planning algorithm;
- how overlapping predicates/states are represented;
- whether planner cost is ever needed;
- whether whole transitions may execute in parallel, beyond the concurrency within one body that §25.2 allows;
- how planner cycles and impossible goals are diagnosed;
- exact lifetime and scoping rules for transition result objects.

## PROACTION

Open research and design:

- its execution model, and what grammar it needs at all;
- rich, persistent and larger-than-memory data, queries and indexes, and the status of `graph` (§34.4);
- the machine-readable boundary between PROACTION and REACTION, and which states may be requested (§34.5);
- mechanisms for communicating with outside systems, and the transport for any off-board link (§34.5, §34.6);
- how PROACTION is given memory on RV-9, and which targets it runs on (§34.4, §34.6);
- learning and adaptation: what may change at runtime, and the path by which stable behavior becomes REACTION source (§37);
- uncertainty and probabilistic reasoning;
- ordering pursuits by importance when they contend, and whether starvation needs an answer (§34.3);
- how concurrent computation is expressed, and how its results return to the authority (§34.3);
- authority domains: how they are declared, when a pursuit's domains are known, and how they are acquired without deadlock (§34.3);
- the historical log: whether anything is summarised as it is discarded, what a power failure costs, entry identity for later metadata, and how it is searched (§34.4);
- whether PROACTION gets dynamically sized arrays, and the bounded heap RV-9 would have to grow for them (§34.4);
- what bounds the interval before a decision point is reached, and the form of the idle fallback (§34.3);
- long-term memory and context representation;
- which ideas from Conatus or other architectures are worth adopting (§35).

## Types

- the spellings of `text[n]`, `instant` and the `deg` literal (§4.1, §5.1);
- whether `fixed[a,b]` is still warranted, once it is known whether the first targets have hardware floating point (§4);
- dimensions and linear algebra: units live on vectors and quantities rather than on arrays, which fits a rotation matrix and a Jacobian badly. Whether anything better exists is open (§4.2);
- absence: R9 has no `optional<T>`, and handles absence per mechanism — a sequence number, an input's default, a published fault. Once parametric types exist one will be proposed, and the answer should be decided rather than drifted into (§44.3);
- text that stores correctly may still not render, where a target's console font covers only ASCII (§4.1).

## Libraries

- the provisional module syntax of §44.1, and whether a library may also export component templates rather than only functions and types;
- whether a library may define operators for its own types (§44.1);
- parametric types: how they are written, and confirming they are monomorphised at compile time (§44.3);
- whether a publication, a log attribute and an estimate are one concept or three (§44.3);
- coordinate frames as a checked property, and how a frame is declared and converted (§44.3);
- how a library carries its layer colouring outward to a program that uses it (§44.3);
- device binding, which decides both hardware naming and simulated substitution (§44.3).

## Federation

- the spelling of an optional component, admitted when its device appears (§45.1);
- how a peer's presence, and a federation's membership, are discovered and published (§45.2);
- how clock offsets are established, exchanged and bounded, and whether a drift bound is declared per machine (§45.4);
- whether RV-9's `watches` needs an optional form, since a watcher of an optional or remote publication must be admittable while its publisher is absent (§45.1);
- whether proof obligations become per-configuration, and whether the degraded configuration — nothing optional present — is the one that must be proven safe (§45.1).

## Proofing

Proofing is a later phase (§33). Until then, decisions in every layer should keep strong compile-time analysis possible.

---

# 43. Current Working Summary

R9 has a relatively mature REALTIME foundation, a REACTION layer that is becoming well defined, and a PROACTION layer under active design.

### REALTIME

```text
realtime
every
on
minimum_interval
deadline
placement
input
expose
limit
failsafe
fault
on stop
```

It executes periodic or event-released physical behavior predictably and publishes coherent observations. It leaves the physical world defined through declared failsafes when a component fails. It compiles its resource requirements into contracts that RV-9 can admit, enforce, and measure.

### REACTION

```text
watch
state
transition
require
in sequence
in parallel
```

It observes published reality, including silence where a publication was expected. Within a transition, work may proceed concurrently wherever the compiler can show the branches do not interfere. It uses known transformations to move the system toward declaratively defined states, which may be parameterized. A transition is eligible only while its requirements hold. REACTION outranks PROACTION, and planning after preemption starts from newly observed reality.

### PROACTION

PROACTION decides what should happen next, on its own initiative and on the system itself, and it may use external intelligence as an optional resource. It reaches the physical world by requesting states and observing results. Its decisions and its changes to authoritative state pass through one serialized authority, while the computation behind them may run concurrently and several pursuits may proceed at once over independent authority domains. Its memory of what it has been through is what REACTION deliberately lacks: records, fixed arrays, and one bounded log of timestamped text with attributes. Beyond the two block forms it has no syntax yet.

The next major design question is PROACTION's execution model, and in particular how to:

> **Make large, persistent, searchable knowledge feel like ordinary data.**


---

# 44. Libraries, and What They Require of the Language

**Recorded 2026-09-18.** R9 has no libraries, and will not have them soon. But the shape of the ones it will eventually need already settles several language questions, and it is cheaper to decide them now than to discover them from twenty libraries pulling in different directions.

## 44.1 Functions, Modules and Names

**Settled; syntax open.**

- **Functions are ordinary declarations, grouped in modules.** A module is a namespace. Related things stay together because they are declared together, not because they hang off a receiver.
- **Methods are not the norm.** R9 deliberately replaced *call the object* with *write an input, read a publication* (§17–§19). A component is reached through its declared inputs and publications and never by invocation. Making methods the usual way to organise code would invite `MOTOR.stop()`, and undo the thing the architecture is built on.
- **Uniform call syntax.** `x.f(y)` means `f(x, y)` where `f` is in scope. That gives chaining and discovery — `v.normalize().scale(2m)` — without an object model, and without having to decide whether `distance` belongs to the first point or the second. Symmetric operations stay symmetric.
- **Operators carry the rest.** `transform * point`, quantity arithmetic, vector algebra. Whether a library may define operators for its own types is open.
- **No inheritance, and no dynamic dispatch hierarchies.** They would defeat the static analysis the lower layers rest on, and §39's rule against vocabulary growth applies.

### Provisional syntax

```text
module geometry

    const TAU = 6.283185307179586

    export function distance(a: position, b: position) returns length
        return magnitude(b - a)
    end

    function magnitude(v: position) returns length      # private
        ...
    end

end
```

```text
use geometry                 # geometry.distance(a, b)
use geometry.distance        # distance(a, b)
use geometry as geo          # geo.distance(a, b)
```

- **`function`** joins the family of single lowercase declaration words: `realtime`, `state`, `transition`, `watch`, `failsafe`.
- **`returns`**, not `->`. The arrow already means a transformation between states (§25), and one symbol should not carry two unrelated meanings. `returns` also ties to the `return` statement, and would extend naturally to several results, where a symbol would need brackets and a tuple type R9 does not have.
- **`name: type`** reuses the colon of §23.2's state parameters.
- **Private by default; `export` publishes.** The instinct is a component's, whose values are private until exposed. `expose` and `export` stay separate words: one is a runtime publication with coherence semantics, the other compile-time visibility.
- **An exported function declares its result type; a private one may leave it inferred** (§6). A library's interface is what colouring and error messages hang off, and should be explicit; a local helper should not need ceremony.
- **Module names are dotted and hierarchical**, and a file declares one module, which lines up with the bundle.
- **A module name is a compile-time namespace. A path is a runtime address.** They are deliberately not made to resemble one another.

Five words — `module`, `use`, `function`, `export`, `returns` — and the largest addition to the vocabulary since the reactive layer. §39's rule is why it waited: libraries are the concrete problem that cannot be expressed without them.

## 44.2 Function Values

**Settled.**

The pressure for first-class functions is not style. It is interchangeable implementations: three IMU drivers behind one interface, or a simulated motor standing in for a real one.

- **In REALTIME and REACTION, implementations are bound when the bundle is built.** Which driver, and whether a device is real or simulated, is a property of the build, like device binding itself. Calls stay direct, so §14.1's colouring stays exact, and substitution still happens.
- **In PROACTION, function values are permitted.** It is the open world. Because a program is compiled as a bundle, the compiler can compute the possible targets of an indirect call, and take a property to hold only where it holds for every one of them.
- **Closures that capture and allocate stay out of REALTIME** (§4).

## 44.3 What the Libraries Demand

Four things R9 does not have, each forced by libraries that will certainly exist.

**Parametric types.** `estimate<T>`, matrices, containers. Monomorphised at compile time and never runtime-polymorphic, so that static analysis and bounded execution survive. This is the largest addition on this list, and it is not designed.

**A value together with what is known about it.** Three places already converge on this shape. A publication carries a value, a sequence number and an observation time (§18.1). A log attribute carries a value and its origin (§34.4). Sensor fusion wants a value, its uncertainty, its time and its provenance. Whether that is one concept or three coincidences should be decided before three libraries each invent their own.

**Coordinate frames, checked like dimensions.** A position in the base frame and one in the camera frame have identical dimensions and must never be added. R9 already checks physical meaning (§5); a frame is the same idea one level up. Kinematics, mapping, localization and vision all depend on getting it right, and the bug it would remove is the kind that produces plausible numbers rather than obvious nonsense.

**Layer colouring, carried outward.** A library function is usable in some layers and not others: geometry is real-time safe and allocation-free, planning and vision are not. §14.1 already derives such properties through the call graph, and a library must carry them to whatever uses it, the way RV-9's target profile marks which of its own calls are real-time safe (§2.1). Without that, the first library call inside a control loop quietly breaks admission.

**One existing open item is promoted.** Device binding is no longer a loose end. It is the mechanism by which a simulated device stands in for a real one, which is what allows autonomy to be tested without launching the machine at a wall.

## 44.4 What a Library May Not Do

**A library acquires no authority its caller lacks** (§31.2). This holds for a learned policy as much as for anything else: *learning a policy is not authorization to execute an action*. A learner proposes; REACTION still decides whether the request is legal; REALTIME still holds its own limits and applies its own failsafes when the policy turns out to be creatively stupid.

**A library may not become a theory of intelligence by the back door.** A behavior or planning library is optional by construction, and nothing below PROACTION may depend on one (§34.1).

---

# 45. Dynamic Hardware and Federation

**Recorded 2026-09-19.** The design has assumed a fixed machine: devices are claimed at admission, implementations are bound when the bundle is built, and `watches` is refused when nothing on the machine could publish the name. Hardware that arrives and leaves while the machine runs, and federations that machines join and leave, break those assumptions — though not, as it turns out, the architecture.

## 45.1 Presence Is Reality

**Settled in principle; spellings open.**

> The set of things that *could* exist is static. The set of things *present* varies, and presence is published reality.

- **Presence is a predicate.** `require ARM.present` makes every transition needing that arm ineligible while it is absent, so the planner routes around it, or fails with `PRECONDITION_FAILED` naming the missing hardware rather than an opaque `NO_PATH` (§25.1). No new mechanism is required.
- **Removal is already a fault.** RV-9 ties device ownership to process lifetime, faults with `DEVICE`, applies the declared failsafe and publishes the fault into the component's cell (§15). A transition awaiting that component fails `COMPONENT_FAILED`, and a `watch` on `.faulted` responds.
- **Addition is admission.** RV-9 loads modules at runtime already. What R9 lacks is a way to declare a component **optional**: compiled into the bundle, admitted when its device appears, absent until then. That is the one new language concept here, and it keeps the graph static — every possible component and transition is known at compile time, and only liveness varies.

## 45.2 Authority Does Not Cross Machines

**Settled.**

REALTIME and REACTION stay machine-local. REACTION's authority rests on bounded planning over a static graph, reading local cells — a snapshot that cannot fail. Across a network none of that survives: latency is unbounded, packets are lost, and partitions happen.

Each machine therefore owns its own hardware, and **a remote peer is a PROACTION-level requester**. It may request states and observe publications; it may never write another machine's inputs. §31.2 already forbids that of PROACTION locally, so federation grants no new privilege — it only puts a network in front of the same boundary. What follows is the property worth having:

> A partition cannot make a machine unsafe. It can only make it idle.

**Contention generalises.** Two machines wanting the same dock, doorway or workpiece is two pursuits wanting the same authority domain (§34.3), one level out. The machine that owns the hardware hosts the arbitration: peers ask, one wins, and losing is an ordinary attempt result (§29). No election, no distributed lock.

## 45.3 Two Things Federation Must Not Do

**The path may span machines; the type must not lie.** RV-9 makes a remote publication addressable exactly as a local one, and that transparency is a trap. A local cell read is a copy that cannot fail; a remote one is a round trip that may be stale, partitioned or gone. A remote observation must carry its age and its absence where a program can see them. This is §44.3's *value together with what is known about it*, and federation is what forces it to exist.

**The graph never assembles itself from the network.** Presence may vary; the set of possible states and transitions stays compiled. A graph built from whatever joined the network cannot be analysed, and with the analysis goes the reason REACTION outranks PROACTION at all.

## 45.4 Time Across Machines

**Settled in direction; provisional.**

- **REALTIME and REACTION never depend on federated time.** Deadlines are local and measured by the local clock. Nothing about a network may reach a control loop.
- **A remote timestamp is converted at the boundary** into local time, using the offset held for that peer, and it carries the **uncertainty** of that conversion: the error in the offset estimate, plus drift since it was last exchanged.
- **Partition degrades precision, not truth.** With no exchange, uncertainty grows at the drift bound. A remote observation becomes less precise — something a program can reason about — rather than wrong, which is something it cannot.
- **Ordering within a machine is exact; ordering across machines is only as good as the offsets.** Where two events from different machines cannot be ordered, the log says so rather than inventing an order, which is the honesty §34.4 already requires of curtailment.

A synchronisation service may narrow the offsets. That is a library (Appendix A), not a language feature.

---

# Appendix A: Library Roadmap

**A roadmap, not a commitment.** These are the libraries R9 will eventually want, or ones like them. They are recorded so that today's decisions are made with tomorrow's weight in mind — §44 exists because of this list. Nothing here is designed, and the order matters more than the contents.

### Tier 1 — foundations

Almost everything else depends on these, and writing them is what forces §44.3's language questions.

| library | contents |
| --- | --- |
| **Math, geometry and quantities** | vectors, matrices, quaternions, transforms, coordinate frames, interpolation, numerical integration, statistics, and R9's unit-aware quantities |
| **Signals and filtering** | moving averages, low/high/band-pass filters, FFT, convolution, Kalman, extended and unscented Kalman, complementary filters, noise models — the plumbing between noisy reality and usable state |
| **Probability and Monte Carlo** | distributions, sampling, Bayesian updates, particle filters, simulation, confidence, probabilistic state — foundational, rather than an add-on to navigation |

### Tier 2 — the machine's own senses and motions

| library | contents |
| --- | --- |
| **Sensors and fusion** | IMUs, accelerometers, gyros, magnetometers, encoders, GNSS, rangefinders, LiDAR, cameras, microphones, temperature, pressure, current. Fusion yields not a bare value but an estimate: value, uncertainty, time, provenance |
| **Control** | PID, feed-forward, state-space, trajectory following, motor control, stabilization, constraints, saturation, later model-predictive control. Binds tightly to REALTIME |
| **Kinematics and dynamics** | forward and inverse kinematics, rigid-body transforms, joint models, differential drive, Ackermann, mecanum and omnidirectional bases, manipulators, humanoid chains, centre of mass, forces and torques |

### Tier 3 — knowing where it is, and what is around it

| library | contents |
| --- | --- |
| **Localization and navigation** | odometry, dead reckoning, particle localization, waypoints, A*, Dijkstra, D*, RRT and RRT*, trajectory generation, obstacle avoidance, probabilistic navigation |
| **Mapping and spatial representation** | occupancy grids, cost maps, point clouds, voxel maps, landmarks, coordinate frames, map merging, eventually SLAM — constrained on small targets, richer on large ones |
| **Motion and manipulation planning** | collision checking, configuration spaces, reachability, grasp planning, trajectory optimization, whole-body movement. Expressed as *put the gripper there*, not *set joint 3 to 27 degrees* |

### Tier 4 — perception

| library | contents |
| --- | --- |
| **Computer vision** | image buffers, camera calibration, resizing, filtering, edges, contours, optical flow, feature detection and matching, object tracking, depth and stereo, hooks for neural inference |
| **Machine perception** | object detection and identity, pose estimation, gesture recognition, face detection and recognition, semantic classification, scene understanding, multimodal perception. Kept above vision, so that *faces* are never baked into the foundations |
| **Audio and speech** | microphone arrays, filtering, direction of arrival, sound classification, wake words, interfaces to recognition and synthesis. Heavy models run elsewhere; RV-9 keeps the real-time audio path |

### Tier 5 — deciding

| library | contents |
| --- | --- |
| **Behavior and planning** | goals, plans, actions, preconditions and postconditions, alternatives, search, utility and cost, hierarchical planning — cooperating with PROACTION rather than replacing it, and optional by construction (§44.4) |
| **Learning** | reinforcement learning, online adaptation, learned models, policy execution, lightweight training. Learning a policy is not authorization to execute an action |

### Cross-cutting — wanted at every tier

| library | contents |
| --- | --- |
| **Fault detection and self-healing** | health observations, anomaly detection, watchdog strategies, degraded modes, redundancy, recovery policies, retry and backoff, substitution, diagnosis. R9's `watch` and `transition` should make this unusually direct |
| **Resource and energy** | CPU budget, memory, battery state, power draw, thermal state, computational cost — so a machine can ask whether an action is *affordable*, not merely possible |
| **Communications and distributed robotics** | CAN, UART, SPI, I²C, networking, telemetry, serialization, discovery, robot-to-robot coordination. Fleets and swarms belong here, never in the RV-9 kernel |
| **Simulation and digital twins** | the same component driving a simulated motor as drives a real one, and sensors interchangeable between physical and simulated sources (§44.3, device binding) |
| **Logging, recording and replay** | record sensor inputs, decisions, transitions and outputs, then replay the world deterministically. PROACTION's log, its recorded commit order (§34.3) and its relative-time views are most of the mechanism already |
| **Visualization and instrumentation** | live plots, maps, trajectories, sensor views, state and transition diagrams, object inspection, resource meters, logs and controls, drawn through RV-9's `/w0` and interactive where it can be |

### Sequence

Tier 1 first, and not only because the rest depend on it: writing it is what settles parametric types, operators and coordinate frames. In that sense the first library is a language decision wearing a library's clothes.
