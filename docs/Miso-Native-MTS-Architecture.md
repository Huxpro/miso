# Miso Native main-thread runtime architecture

Status: local RFC / measured decision record. This document does not select or
implement a new default.

## Summary

The current native build imports the same pre-bundled GHC/Miso program into
both Lynx scripting layers. This preserves synchronous first rendering and
app-specific main-thread handlers, but it also puts a substantial general
Haskell runtime in the main-thread script (MTS).

A measured core fixture ships 9,592,648 bytes, of which 4,080,853 bytes are MTS
bytecode. An experimental fixed TypeScript executor reduces the same fixture to
5,522,973 bytes (-42.4%) and its MTS bytecode to 9,709 bytes (-99.8%). That
experiment is intentionally not proposed as equivalent: it changes first paint
to asynchronous BTS patches and cannot execute app-specific `onMain` handlers.

The decision is therefore architectural, not a bundler flag. This RFC asks
maintainers to choose an explicit semantic mode before product code is added.

## Why Rspack tuning is not the pivot

`mkLynxBundle` first asks Bun to pre-bundle the GHC output into one opaque
`all.js`. Rspeedy then imports that module into both `react:main-thread` and
`react:background` layers. An instrumented non-concatenated build found:

| Layer | Module | Transformed size | Import reason |
|---|---|---:|---|
| `react:main-thread` | `./all.js` | 10,196,952 B | `./entry.js` import |
| `react:background` | `./all.js` | 10,197,324 B | `./entry.js` import |

Rspack sees no Haskell module graph below `all.js`, so module concatenation,
split chunks, or ordinary tree shaking cannot produce a meaningfully thin MTS.
The split must happen before this pre-bundle, either by selecting a distinct
entry/program or by generating a main-thread slice.

## Runtime constraints

Lynx uses distinct main-thread and background runtimes. The main-thread runtime
participates directly in first-screen rendering, Element PAPI operations and
synchronous main-thread event handling; the background runtime runs application
logic and later tree updates. See the public Lynx documentation:

- <https://lynxjs.org/guide/scripting-runtime/index.html>
- <https://lynxjs.org/3.8/react/lifecycle.html>
- <https://lynxjs.org/3.6/react/main-thread-script>
- <https://lynxjs.org/3.5/guide/scripting-runtime/main-thread-runtime>

Miso currently uses the full app on MTS for more than patch execution:

1. It synchronously renders the initial tree.
2. It knows the ordinary BTS event-name manifest used to install delegation.
3. It resolves app-specific `StaticPtr` identities for `onMain` handlers.
4. It decodes those events at the component action type.
5. It mirrors component model, props and context for synchronous handler reads.
6. It participates in component READY/ACK, mount and unmount messages.

A small universal executor can implement Element PAPI and patches, but it
cannot infer the app-specific items above from the current patch stream.

## Measured control

The control routes `all.js` to `react:background` only and gives
`react:main-thread` a pre-built TypeScript patch executor. It also forces the BTS
to send the initial tree as patches and supplies a temporary explicit `tap`
manifest.

| Invariant | Full runtime | Fixed thin control |
|---|---|---|
| shipped core bundle | 9,592,648 B | 5,522,973 B |
| MTS bytecode | 4,080,853 B | 9,709 B |
| first paint | synchronous MTS render | asynchronous BTS patch batch |
| ordinary BTS event | pass | pass with explicit event manifest |
| declarative update | pass | pass |
| remove/reinsert churn | pass | pass |
| nested BTS component state | pass | pass |
| MTS-only style handler | pass | unavailable |
| hydrated model read on MTS | pass | unavailable |
| app-specific main-thread dispatcher | present | absent |

This establishes a size ceiling and a viable BTS-driven subset. It does not
establish startup, memory, synchronous first paint, lifecycle, or `onMain`
equivalence.

## Options

### A. Keep the full Miso/GHC runtime on MTS

Semantics remain unchanged. Work focuses on compiler/link reachability and on
preventing accidental MTS growth.

Advantages:

- synchronous first paint and current `onMain` behavior already work;
- one app program defines handler types and state on both threads;
- smallest migration and debugging risk.

Costs:

- about 4.08 MB of MTS bytecode for the measured core fixture;
- a general GHC runtime is parsed/initialized in the pixel pipeline;
- the current pre-bundle prevents downstream module-level elimination.

Before accepting the cost as the default, add decoded-section reporting and a
budget, then obtain startup/parse/memory traces from a trace-capable Lynx host.

### B. Add an explicit BTS-only thin mode

This mode formalizes the measured control as a different product contract:

- first paint is asynchronous and BTS-driven;
- `onMain` is rejected at build time or explicitly rerouted asynchronously;
- build output includes an event manifest;
- the common MTS executor owns Element PAPI, patches and event forwarding only.

This is the lowest-complexity path to the measured size reduction, but it must
be opt-in and named as a semantic mode. It must not silently change existing
applications.

### C. Generate a common executor plus an app-specific MTS slice

The common executor owns PAPI, patch application and protocol machinery. A
generated app slice contains only:

- the synchronous first-frame program or data needed to reproduce it;
- the event-name manifest;
- referenced `onMain` handlers and their decoders;
- the minimal component model/props/context codecs and state transitions those
  handlers read;
- stable handler/component identity tables.

This option targets current semantics without the full runtime, but it creates
a compiler/linker boundary and a versioned wire protocol. It is the highest
engineering cost and must prove that the generated slice remains debuggable and
smaller for real applications, not only the core fixture.

## Required wire contract for B or C

Any thin mode needs a versioned contract independent of timing accidents.

### Build metadata

- protocol and executor version;
- supported semantic mode (`bts-only` or `generated-slice`);
- complete delegated event-name/phase/direct-binding manifest;
- component and handler identity table;
- capability bits for synchronous first paint and `onMain`;
- hash binding metadata, BTS program and MTS slice to the same build.

### First frame

- explicit root/session identity;
- `READY` before initial work can be delivered;
- a structural first-frame manifest and ACK/NACK boundary;
- no incremental patch or main-thread event release before adoption;
- duplicate, stale and reordered messages are inert;
- bounded queues and a timeout/recovery policy.

The first-frame reconciliation work is orthogonal to the size choice. Its state
machine can be reused, but a BTS-only mode cannot claim synchronous IFR merely
because it uses the same protocol.

### Components and state

- mount/unmount with generation/session identity;
- model, props and context snapshot versions;
- READY/READY_ACK and disposal acknowledgement;
- ordering rules between hydration, an event and a BTS update;
- stale component messages rejected after unmount/reload;
- bounded retention for snapshots and pending effects.

### Events and handlers

- build-time failure for unsupported synchronous handlers in BTS-only mode;
- target node, phase, direct/delegated mode and propagation result;
- typed decoder/handler lookup in generated-slice mode;
- defined effect ownership and reentrancy behavior;
- diagnostics for missing IDs, codecs or state generations.

## Decision matrix

| Criterion | A: full MTS | B: BTS-only | C: generated slice |
|---|---|---|---|
| current first-paint semantics | yes | no | target: yes |
| current `onMain` semantics | yes | no / async fallback | target: yes |
| measured size reduction | none yet | -42.4% core | unknown, expected between A/B |
| compiler changes | low | low/moderate | high |
| runtime protocol complexity | current | moderate | high |
| migration risk | low | explicit opt-in | moderate/high |
| reversible rollout | yes | yes, mode flag | yes, retain A fallback |

## Gradual validation and rollout

Do not make a thin mode the default during initial development.

1. **L0 — required:** protocol/state-machine/unit tests, build metadata
   validation, unsupported-handler diagnostics, stale/duplicate/reordered input,
   queue bounds and deterministic codec/identity tests.
2. **L1 — required:** deterministic two-context simulation covering initial
   frame, event manifest, READY/ACK, mount/unmount, model/props/context updates,
   synchronous handler reentrancy where supported, and reload.
3. **L2 — initially non-blocking:** Lynx testing-environment runtime tests. Make
   required only after the cases and runtime version are stable.
4. **L3 — scheduled/manual first:** same-device full-vs-thin Sandbox matrix for
   first paint, ordinary events, `onMain`, dynamic churn, nested components,
   List, reload and fault injection.
5. **Cost gate:** decoded shipped sections on every build; startup/parse trace
   and attributed memory on a trace-capable `local_test` host. Do not substitute
   one-shot process PSS for attributed evidence.
6. **Canary:** explicit application opt-in with A retained as fallback. Promote
   only after compatibility and performance budgets hold across representative
   applications.

## Questions for maintainers

1. Is preserving synchronous first paint a non-negotiable property for every
   Miso Native app, or may an explicit BTS-only mode choose async first paint?
2. Must `onMain` preserve synchronous typed access to component state, or is an
   async BTS fallback an acceptable separate API?
3. Should option A remain the default while B/C are experimental, with a build
   budget preventing further MTS growth?
4. For option C, should the slice boundary be generated from `StaticPtr`
   reachability, an explicit user annotation/manifest, or both?
5. What mismatch recovery should hosts expose: freeze with fatal diagnostic,
   root reload/remount, or an application callback?

## Proposed next experiment

Prototype metadata and protocol validation before another executor. The next
code change should generate an event/capability/handler manifest and fail a
BTS-only build containing `onMain`. Once that contract is stable, compare:

- B, using the already measured common executor; and
- a minimal C slice containing the core fixture's first frame and two
  app-specific main-thread handlers.

Run both against the same conformance bundle and device matrix. Keep this work
independent of the IFR contribution so maintainers can accept first-frame
correctness without choosing an MTS architecture.
