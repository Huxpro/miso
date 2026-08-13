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
The 4,080,853-byte MTS chunk is already PrimJS bytecode: source parsing and
minification headroom are already spent at build time, leaving load,
initialization, memory and bytecode evaluation costs. Only changing what is
linked into the MTS can materially move those costs.

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

### B+. Add a build-time first-frame manifest to the thin mode

Synchronous first paint and app-specific `onMain` are separate problems. First
paint can be represented as data: run the BTS program once at build time,
record its slot-normalized first-frame manifest and embed that data for replay
by the common MTS executor. The runtime BTS then validates its own initial frame
against the embedded manifest using the same `compareInitialFrames` contract.

This restores synchronous first paint while keeping MTS bytecode near B's KB
range. `onMain` remains unavailable or explicitly async-degraded, so B+ is a
complete terminal mode for applications that do not use synchronous main-thread
handlers. It is also a lower-risk staging point for C: the existing runtime
recorder already emits the required manifest format; an offline Node run closes
the build-time recording → generic executor replay → runtime BTS validation
loop.

The prototype must prove deterministic build inputs (global props, locale,
clock/randomness and native capability reads) or reject pages whose first frame
cannot be recorded reproducibly.

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

Its first feasibility gate is `StaticPtr` identity. `StaticKey` fingerprints
must remain identical when the same closure is linked into two independent GHC
JS programs (the full BTS app and the generated MTS slice). If they differ by
link unit, the build must generate and validate an explicit cross-program key
mapping table. Test this before investing in reachability-driven slicing.

## Required wire contract for B or C

Any thin mode needs a versioned contract independent of timing accidents.

### Build metadata

- protocol and executor version;
- supported semantic mode (`bts-only`, `bts-manifest` or `generated-slice`);
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
| measured size reduction | none yet | -42.4% core (shipped size only; startup/memory not yet measured) | unknown, expected between A/B |
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
5. After the safe default BTS-authoritative repaint, should hosts additionally
   expose root reload/remount, a fatal diagnostic or an application callback?

## Prototype results

Two deliberately small experiments were run against the same GHC JavaScript
toolchain and the conformance fixture used by the first-frame work.

### `StaticPtr` link-graph stability

The identical `Shared.hs` closure was compiled into two independent GHC JS
programs. The minimal program linked only that closure; the full program added
`Data.Map`, 200 map entries and an unrelated static closure. Their generated
`all.js` files were 2,168,273 and 2,297,881 bytes respectively, so the final
link graphs were observably different. Both programs printed the same shared
key:

```text
9f8082e547c1407882e17e3eaf2f05e5
```

The full program's unrelated closure had a distinct key
`3e1241819ab6d1a7f6669a9e78ccf059`. This removes simple link-set membership as
an immediate blocker for C. It does **not** prove stability across source moves,
package/unit-id changes, compiler upgrades or transformations that clone the
closure. A generated slice should still emit and validate a BTS↔MTS key table
at build time rather than treating this one result as an ABI guarantee.

### B+ offline first-frame replay

The exact conformance BTS executable was run twice under Bun with a mock Lynx
peer, using the production recorder rather than a hand-written fixture. Each
run produced a slot-normalized manifest with 29 nodes and 109 operations:

| Property | Result |
|---|---:|
| compact manifest JSON | 11,917 B |
| SHA-256, run 1 | `76a734aba48f34b1ed7072d5930e744b6a39e4ec2bef6d6900296146e7e926b5` |
| SHA-256, run 2 | `76a734aba48f34b1ed7072d5930e744b6a39e4ec2bef6d6900296146e7e926b5` |
| generic executor replay | 29 nodes, one root child, 18 event bindings |
| serialized replay tree | 7,774 B; identical for both runs |

The executor reconstructed the expected IDs, text, styles, hierarchy and event
identity records. This is concrete evidence that B+ can restore this fixture's
synchronous first-frame *shape* as data without linking the Haskell runtime on
MTS. It also demonstrates the boundary: the 18 event records name handlers but
do not contain their app-specific implementation or typed state access, so the
experiment does not solve `onMain`.

This closes only the static-fixture determinism and generic-replay gate. A real
B+ implementation must still reject or parameterize first frames that depend
on global props, locale, clock/randomness or native capability reads, embed the
manifest into a real Lynx MTS bundle, and validate it against the runtime BTS
digest on device.

## Remaining experiment and rollout

Next, generate an event/capability/handler manifest and fail a BTS-only build
containing `onMain`. Once that contract is stable, compare:

- B, using the already measured common executor; and
- B+, using the offline first-frame manifest; and
- a minimal C slice containing the core fixture's first frame and two
  app-specific main-thread handlers.

Run both against the same conformance bundle and device matrix. Keep this work
independent of the IFR contribution so maintainers can accept first-frame
correctness without choosing an MTS architecture.
