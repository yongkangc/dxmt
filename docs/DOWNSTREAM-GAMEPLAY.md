# Downstream gameplay fixes

`yongkangc/dxmt`'s `gameplay-performance` branch contains experimental changes
based on Sikarugir-App/dxmt `e94c312f5c054263acf261cfa109edf13e757587`.
This is AI-assisted work requested for the user's own fork. Upstream's
`AGENTS.md` prohibits AI-generated contributions; no upstream submission is made.
Existing source licenses and copyright notices are preserved.

## Changes

- Dynamic buffer/texture discard maps invalidate only bound vertex buffers,
  constant buffers and SRVs referencing the renamed resource. SRVs compare their
  underlying resource, so multiple views are handled. Existing dirty bits and
  whole-set invalidation at encoder/state transitions are preserved.
- Compiler completion publishes and notifies only when it has dependent work:
  one notification for one continuation, a broadcast for multiple continuations.
  The processor-based worker cap and requested priority are unchanged.
- Scheduler shutdown sets its wait predicate while holding the worker mutex,
  preventing a lost notification between an idle worker's predicate check and
  wait. Shutdown still broadcasts and joins workers.
- Ordinary graphics pipelines normalize index format on a descriptor copy.
  Geometry/tessellation cache methods retain their index-dependent variants.
- Pipeline hashing and equality include stream-output layout identity.
- Non-debug builds skip the debug HUD's per-frame string formatting. Debug HUD
  output is preserved.

The second hot-path audit also fixes these state transitions:

- Unchanged vertex-buffer binds queue no offset command, and unchanged input
  layouts leave the ready pipeline intact.
- Replacing a bound slot updates its hazard bit in both directions. Hazard
  resolution now sees a hazardous resource replacing a read-only resource.
- Unbinding reflected UAVs refreshes their argument table, including when the
  last UAV is removed. OM dirtiness is cleared after all active graphics stages
  have observed it, including draws without a pixel shader. Access tracking for
  still-bound UAVs remains per use.
- Whole-buffer constant-buffer calls reset a previous range's offset and size
  metadata. Unbound getter outputs are initialized.
- Geometry/tessellation ready-pipeline checks account for 16-/32-bit and
  non-indexed draw variants. Ordinary draws do not need this extra check.
- The dynamic-map invalidation helper is shared with deferred contexts, and
  deferred Map checks null arguments before resource access.

## Validation

The regression runner and fixtures live in the companion
[Sikarugir fork](https://github.com/yongkangc/Sikarugir):

```sh
python3 /path/to/Sikarugir/tools/validate_dxmt_gameplay.py --source . --repeat 10
python3 /path/to/Sikarugir/tools/validate_dxmt_gameplay.py --source . --architecture x86_64 --repeat 10
python3 /path/to/Sikarugir/tools/validate_dxmt_gameplay.py --source . --sanitize --repeat 5
python3 /path/to/Sikarugir/tools/audit_dxmt_hotpath.py --source .
python3 /path/to/Sikarugir/tools/audit_dxmt_hotpath.py --source . --architecture x86_64
python3 /path/to/Sikarugir/tools/audit_dxmt_hotpath.py --source . --sanitize
```

These commands compile the actual BindingSet and scheduler templates, extracted
invalidation helper, pipeline descriptor/hash/equality and ordinary/geometry/
tessellation cache methods. Resource, Metal and Win32 objects are substitutes;
workers use native threads, including under Rosetta for x86-64. The fixture checks
all three Map invalidation call sites, multiple views/stages, high SRV slots,
unbound/replaced resources, preserved prior dirty bits, semantic blend equality,
separate stream-output layouts, single/fan-out/chained/multiple dependencies,
completion-registration races, concurrent producers and idle shutdown.

A shutdown fixture holds the wait lock briefly after a false predicate check,
simulating preemption before the actual wait. The prior destructor timed out in
this window; the mutex-protected shutdown passes. This does not establish the
cause of any observed game hang.

No complete Wine/Metal engine was built or installed. These are source regression
checks, not shader-output or game-rendering validation. A full build requires the
toolchain in [DEVELOPMENT.md](DEVELOPMENT.md). Next, use an isolated test wrapper
and compare fixed scenes with each change separately, checking output correctness
and repeated median/p95/p99 frame intervals before claiming gameplay gains.
