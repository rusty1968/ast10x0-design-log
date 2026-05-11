# Decision Record: Board flash config as single source of truth

- Decision ID: DEC-20260511-board-flash-config-single-source
- Status: accepted
- Owner: platform architecture
- Date: 2026-05-11
- Reviewers: firmware, platform
- Related decisions: DEC-20260509-board-orchestration-layer, DEC-20260510-smc-topology-modeling

## 1. Decision Question

Where should flash device geometry (`FlashConfig`) and controller topology
(`SmcConfig`) be declared, such that the backend, the server binaries, and
future board variants all read from a single authoritative location instead of
each hardcoding their own copies?

## 2. Context And Constraints

- Technical context: flash device geometry (capacity, page/sector/block sizes,
  clock rate) and controller topology (which chip-selects are populated, DMA
  enabled) were declared redundantly — in the flash backend, in test fixtures,
  and implicitly in QEMU model selection — with no single authoritative source.
  The backend layer carried board-specific values, meaning changing a board
  required editing a backend implementation file. The QEMU and production flash
  devices have different capacities; conflating them at build time silently
  produces wrong AHB window mappings on hardware.

  `DEC-20260509-board-orchestration-layer` establishes that a board crate is
  the source of truth for board-specific declarations.
  `DEC-20260510-smc-topology-modeling` adds topology modeling to the SMC
  config. Neither decision reached the flash config layer.

- Security constraints: none specific to this decision; flash geometry errors
  (wrong capacity) produce `InvalidCapacity` errors at runtime, not silent
  data corruption.

- Reliability constraints: a wrong `capacity_mb` causes the segment register
  to be programmed incorrectly, which corrupts the AHB flash window mapping
  silently on hardware. Production and QEMU values must not be confused at
  build time.

- Performance constraints: none. Config is `const`; zero runtime cost.

- Delivery constraints: the `backend/flash` crate must remain `no_std` and
  produce a single Bazel library target.

- Non-goals:
  - Cross-validating `system.json5` process memory sizes against flash device
    geometry. That requires codegen or a shared Starlark constant and is a
    separate follow-on.
  - Multi-CS board topology (cs1) — not required now; the board crate will
    have `cs1: None` initially. Adding a second CS becomes a one-line change
    in the board crate rather than a backend edit.
  - Runtime board detection. Config is compile-time only.

## 3. Options Considered

### Option A: `target/ast10x0/board` crate with Bazel `select()` for QEMU vs production (chosen)

- Summary: a new `rust_library` crate at `//target/ast10x0/board` exports
  named `const` values (`FMC_CS0: FlashConfig`, `FMC_CONFIG: SmcConfig`, etc.)
  built from the `FlashConfig::winbond_*` presets already in `types.rs`. A
  Bazel `select()` swaps in `src/qemu.rs` (1 MB `w25q80bl`) when
  `//target/ast10x0:qemu_enabled` is true, and `src/lib.rs` (8 MB W25Q64)
  otherwise. `backend/flash` removes `const FLASH_CFG` and imports from the
  board crate.
- Benefits: one edit location per board; production vs QEMU split is explicit
  and build-time verified; uses existing `qemu_enabled` config_setting (already
  in `target/ast10x0/BUILD.bazel`); named presets from `types.rs` propagate
  automatically; `server_main*.rs` binaries are unchanged.
- Costs: new crate; `backend/flash/BUILD.bazel` gains one dep.
- Main risks: `select()` on `qemu_enabled` adds an implicit dependency between
  the board crate and the QEMU flag; callers that use `new_with_cfg()` to
  override config (tests) are unaffected but must not rely on `FLASH_CFG`
  being accessible.
- Reversibility: medium — removing the board crate later requires moving
  constants back into the backend or callers.

### Option B: Promote `FlashConfig::winbond_w25q64()` preset directly in the backend

- Summary: replace `const FLASH_CFG` with `FlashConfig::winbond_w25q64()` in
  `new_for_controller()`. Keep QEMU override in `new_with_cfg()`.
- Benefits: no new crate; one-line change.
- Costs: the backend still owns the board declaration; QEMU vs production
  distinction requires a caller-supplied override or a `cfg` flag in the
  backend, not at the board layer. The architectural gap (backend = board
  description) is not closed.
- Reversibility: high.

### Option C: Starlark constants in `defs.bzl` injected as `rustc --cfg`

- Summary: declare flash geometry in `target/ast10x0/defs.bzl` as Starlark
  constants; pass them through `rustc_env` or `--cfg` flags; read them from
  Rust with `env!()` or `#[cfg()]`.
- Benefits: single source shared with the Bazel build graph; could
  theoretically cross-validate `system.json5`.
- Costs: `rustc_env` values are strings; typed `FlashConfig` construction from
  strings requires parsing; error messages are poor; `#[cfg()]` is boolean
  only, not numeric. Significant complexity for what is currently just a few
  `u32` fields.
- Reversibility: low.

## 4. Evidence Summary

- **Backend should not own board identity.** A backend is a transport and
  protocol adapter; it should receive configuration, not define it. When a
  backend hardcodes device geometry, adding a board variant requires editing
  an implementation file rather than a configuration file — inverting the
  dependency.

- **QEMU and production flash devices differ in capacity.** Conflating them
  at the wrong layer produces incorrect hardware register programming. The
  build system already has an explicit QEMU flag; the board crate is the
  natural place to consume it.

- **Named flash presets already exist in the peripheral crate.** The typed
  preset infrastructure (`winbond_w25q64()`, etc.) was unused by anything
  above the peripheral layer. Routing through a board crate lets that layer
  propagate automatically to all consumers.

- **Test fixtures construct SMC config directly and are unaffected.** The
  board crate boundary does not touch the test layer; tests can continue to
  construct configurations inline without going through the board crate.

- **`DEC-20260509-board-orchestration-layer`** declares the board crate as
  the source of truth for board-specific values. This decision closes the gap
  at the flash config layer.

## 5. Decision

**Option A** — new `//target/ast10x0/board` crate with `select()`-based
QEMU/production split.

The architectural gap (backend owns board identity) is real and will only grow
as board variants are added. Option B patches the symptom without closing the
gap. Option C is disproportionately complex.

## 6. Consequences And Follow-ons

- Adding a board variant (e.g., a 32 MB W25Q256 board) becomes an edit to
  `board/src/lib.rs` only; the backend and server binaries are untouched.
- Multi-CS (`cs1: Some(...)`) is a one-line change in the board crate.
- `system.json5` cross-validation remains out of scope. The flash region sizes
  in `system.json5` are process memory layout (kernel concern), not flash
  device geometry (board concern). Bridging them requires a codegen step and
  is a separate decision.
