# Decision Record: Board flash config as single source of truth

- Decision ID: DEC-20260511-board-flash-config-single-source
- Status: proposed
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

- Technical context: flash configuration is currently duplicated and
  inconsistent across three locations:

  | Location | What it declares | Problem |
  |---|---|---|
  | `target/ast10x0/backend/flash/src/lib.rs` — `const FLASH_CFG` | `capacity_mb: 1`, `spi_clock_mhz: 25`, page/sector/block sizes | Hardcoded; ignores `FlashConfig::winbond_*` named presets already in `types.rs`; the 1 MB value matches the QEMU `w25q80bl` model, not the production Winbond W25Q64 (8 MB) |
  | `SmcConfig` constructed in `new_with_cfg()` | `cs1: None`, `dma_enabled: false` | Board topology hardcoded in backend logic; changing a board requires editing a backend implementation file |
  | `system.json5` per test | Process RAM/flash region sizes, IRQ assignments | Declares no flash device geometry; cannot cross-validate against `FLASH_CFG`; a different document of the same board |

  `DEC-20260509-board-orchestration-layer` already establishes that a board
  crate is the source of truth for board-specific declarations.
  `DEC-20260510-smc-topology-modeling` adds `SmcTopology` to `SmcConfig`.
  Neither decision reached the flash config layer.

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

| Source | Supports | Confidence |
|---|---|---|
| `backend/flash/src/lib.rs` `const FLASH_CFG { capacity_mb: 1 }` | Config is hardcoded in backend, not from board | high |
| `types.rs` `FlashConfig::winbond_w25q64()` (8 MB) | Named preset exists and is unused by the backend | high |
| `target/ast10x0/BUILD.bazel` `bool_flag(name = "qemu")` + `config_setting(name = "qemu_enabled")` | QEMU flag infrastructure already exists for `select()` | high |
| `tests/smc/target_*.rs` all construct `SmcConfig` directly | Test layer bypasses backend; board crate change does not affect tests | high |
| `DEC-20260509-board-orchestration-layer` | Board crate is the declared source of truth for board-specific values | high |

## 5. Decision

**Option A** — new `//target/ast10x0/board` crate with `select()`-based
QEMU/production split.

The architectural gap (backend owns board identity) is real and will only grow
as board variants are added. Option B patches the symptom without closing the
gap. Option C is disproportionately complex.

## 6. Implementation Plan

### Step 1 — Create `target/ast10x0/board/`

```
target/ast10x0/board/
    BUILD.bazel
    src/
        lib.rs      ← production board constants
        qemu.rs     ← QEMU model overrides
```

`src/lib.rs` (production):
```rust
use ast10x0_peripherals::smc::{FlashConfig, SmcConfig, SmcController};

pub const FMC_CS0:  FlashConfig = FlashConfig::winbond_w25q64();  // 8 MB
pub const SPI1_CS0: FlashConfig = FlashConfig::winbond_w25q64();
pub const SPI2_CS0: FlashConfig = FlashConfig::winbond_w25q64();

pub const FMC_CONFIG: SmcConfig = SmcConfig {
    controller_id: SmcController::Fmc,
    cs0: Some(FMC_CS0),
    cs1: None,
    dma_enabled: false,
    enable_interrupts: false,
};
```

`src/qemu.rs` (QEMU `w25q80bl` model — 1 MB):
```rust
use ast10x0_peripherals::smc::{FlashConfig, SmcConfig, SmcController};

pub const FMC_CS0:  FlashConfig = FlashConfig {
    capacity_mb: 1,
    page_size: 256,
    sector_size: 4096,
    block_size: 65536,
    spi_clock_mhz: 25,
};
pub const SPI1_CS0: FlashConfig = FMC_CS0;
pub const SPI2_CS0: FlashConfig = FMC_CS0;

pub const FMC_CONFIG: SmcConfig = SmcConfig {
    controller_id: SmcController::Fmc,
    cs0: Some(FMC_CS0),
    cs1: None,
    dma_enabled: false,
    enable_interrupts: false,
};
```

`BUILD.bazel`:
```python
rust_library(
    name = "board",
    srcs = select({
        "//target/ast10x0:qemu_enabled": ["src/qemu.rs"],
        "//conditions:default":           ["src/lib.rs"],
    }),
    crate_name = "board",
    edition = "2024",
    target_compatible_with = TARGET_COMPATIBLE_WITH,
    visibility = ["//target/ast10x0:__subpackages__"],
    deps = ["//target/ast10x0/peripherals"],
)
```

### Step 2 — Update `backend/flash/src/lib.rs`

Remove `const FLASH_CFG`. Replace with:
```rust
use board::{FMC_CS0, FMC_CONFIG, SPI1_CS0, SPI2_CS0};
```

`new_for_controller()` uses `FMC_CONFIG` / `SPI1_CS0` / `SPI2_CS0` directly.
`new_with_cfg()` is kept as the test/override path.

### Step 3 — Update `backend/flash/BUILD.bazel`

Add `//target/ast10x0/board` to `deps`.

### Step 4 — Verify no change to server binaries or tests

`server_main.rs`, `server_main_spi1.rs`, `server_main_spi2.rs` call
`Backend::new_fmc()` / `new_spi1()` / `new_spi2()` — unchanged. Tests that
construct `SmcConfig` directly — unchanged. Run full baseline after the change:

```
bazelisk test //target/ast10x0/peripherals/... && \
bazelisk test --config=virt_ast10x0 --test_tag_filters= //target/ast10x0/...
```

## 7. Consequences And Follow-ons

- Adding a board variant (e.g., a 32 MB W25Q256 board) becomes an edit to
  `board/src/lib.rs` only; the backend and server binaries are untouched.
- Multi-CS (`cs1: Some(...)`) is a one-line change in the board crate.
- `system.json5` cross-validation remains out of scope. The flash region sizes
  in `system.json5` are process memory layout (kernel concern), not flash
  device geometry (board concern). Bridging them requires a codegen step and
  is a separate decision.
