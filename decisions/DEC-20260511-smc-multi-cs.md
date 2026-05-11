# Decision Record: Multi-chip-select modeling in the AST10x0 SMC peripheral

- Decision ID: DEC-20260511-smc-multi-cs
- Status: accepted
- Owner: platform architecture
- Date: 2026-05-11
- Reviewers: firmware, platform
- Related decisions: DEC-20260509-peripheral-ownership-typestate, DEC-20260509-lifetime-bounded-device-facade, DEC-20260510-smc-topology-modeling, DEC-20260511-board-flash-config-single-source

## 1. Decision Question

How should the AST10x0 SMC peripheral model a single controller with multiple
attached flash devices (CS0 and CS1), such that per-CS dispatch, per-CS
configuration, and per-CS capacity validation are all sound — without requiring
callers to manage two separate controller instances or perform unsafe indexing?

## 2. Context And Constraints

- Technical context: the AST10x0 FMC and SPI controllers each expose two
  independent chip-select lines. Each CS has its own segment register (base +
  size), its own SPI control register (frequency divisor, transfer mode), and
  its own AHB flash window region. A single controller instance owns both
  CSes; they share the same MMIO base but program separate register slots.
  aspeed-rust treats the two CSes as a flat array indexed by `u8`.

- Composition with prior decisions:
  - DEC-20260509-peripheral-ownership-typestate: peripheral lifecycle is
    encoded as type-state. The same principle — making invalid states
    unrepresentable — should extend to CS selection.
  - DEC-20260509-lifetime-bounded-device-facade: the `SpiNorFlash` device
    facade is borrowed from `Smc<Ready>` and targets a single CS. The borrow
    prevents concurrent access to the same CS through two facades.
  - DEC-20260511-board-flash-config-single-source: each CS gets its own
    `FlashConfig`; the board crate is the source of truth.

- Constraints:
  - A controller may be populated with CS0 only, CS1 only (unusual), or both.
    The validity of a CS at runtime depends on whether the board wired a flash
    device there, not on a hardware register.
  - Capacity validation must be per-CS: the `SpiNorFlash` facade's geometry
    must match the selected CS's configured capacity, not the controller total.
  - The transport layer (`transceive_user`) must assert the correct CS line;
    both CSes share the user-mode AHB window, but the segment registers
    partition the address space.
  - `no_std`; no heap; controller instances live in static or stack storage.

- Non-goals:
  - More than two chip-selects per controller. The AST10x0 hardware supports
    at most CS0 and CS1 per controller, so the design is intentionally
    constrained to exactly two CS slots. It does not generalize to N CS:
    the config and internal state are structured for exactly 2, and `ChipSelect`
    is a closed 2-variant enum. If a future SoC exposed 3+ CS lines, the right
    generalization would be a fixed-size config array and a validated newtype
    index — effectively Option C from §3 but with a thin safety wrapper.
    That redesign is out of scope here.
  - Runtime hot-plug of a second flash device after `init()`.
  - Separate DMA configuration per CS. DMA is controller-scoped in this
    generation.

## 3. Options Considered

### Option A: `ChipSelect` enum passed at each call site; per-CS `Option<FlashConfig>` in `SmcConfig` (chosen)

- Summary: a `ChipSelect { Cs0, Cs1 }` enum is added to the public API.
  `SmcConfig` carries `cs0: Option<FlashConfig>` and `cs1: Option<FlashConfig>`.
  Every transport and device-facade API (`transceive_user`, `read`, `write`,
  `erase`, `SpiNorFlash::from_fmc_cs`) takes an explicit `ChipSelect` argument.
  The controller validates at call time: if `Cs1` is requested and
  `config.cs1.is_none()`, it returns `SmcError::InvalidChipSelect`.
  Segment registers are programmed for each populated CS at `init()`;
  per-CS AHB window addresses and normal-read control words are stored
  internally and restored unconditionally after every user-mode transaction.
- Benefits: a single controller instance manages both CSes; no ownership
  split; per-CS validation is a runtime check tied directly to the config;
  pattern is identical to aspeed-rust's array indexing but typed; the
  `SpiNorFlash` facade can target either CS via the same borrow of
  `Smc<Ready>`.
- Costs: every transport call carries an explicit `ChipSelect` argument even
  in the common single-CS case; `Option<FlashConfig>` for each CS slot adds a
  small constant overhead to `SmcConfig`.
- Main risks: callers might pass `Cs1` without populating `cs1` in the config;
  caught at call time with `InvalidChipSelect` — not at construction time.
- Reversibility: medium — changing the call-site signature later requires
  updating all callers.

### Option B: Two separate `Smc<Ready>` instances, one per CS

- Summary: each CS is represented by an independent controller instance with
  a CS-specific base address offset or a wrapper that fixes the CS at
  construction.
- Benefits: no runtime CS argument; a CS0-only caller cannot accidentally
  address CS1.
- Costs: two instances sharing the same MMIO base is unsound — both would
  write to overlapping per-CS registers independently, violating the
  single-owner requirement from DEC-20260509-peripheral-ownership-typestate.
  Initialization order between the two instances is undefined.
- Main risks: register aliasing; no way to safely enforce that only one
  instance holds the MMIO at a time without global locking.
- Reversibility: N/A — rejected.

### Option C: Runtime `u8` CS index, no enum

- Summary: callers pass `0` or `1` directly; out-of-range values are checked
  at runtime.
- Benefits: minimal type surface; matches aspeed-rust literally.
- Costs: `u8` carries no semantic information; documentation cannot express
  which values are legal without a comment; type-safety benefit of the enum
  is lost; out-of-range values like `2` are valid `u8` but invalid CS
  indices — the error is caught later than necessary.
- Reversibility: medium.

### Option D: Type-state per CS — `Smc<Ready, Cs0>` and `Smc<Ready, Cs1>`

- Summary: the CS is encoded as a second type parameter; separate instances
  are created by splitting a dual-CS controller into two typed handles at
  `init()`.
- Benefits: CS selection errors are caught at compile time.
- Costs: `init()` must produce two typed values simultaneously (awkward in
  Rust without returning a tuple or a split-result struct); the type system
  does not prevent two callers from each holding a typed handle to the same
  controller; MMIO aliasing risk returns. Adds significant generic complexity
  throughout the call stack.
- Reversibility: low.

## 4. Evidence Summary

- **Single-owner MMIO** (DEC-20260509-peripheral-ownership-typestate): a
  controller's registers are owned by exactly one `Smc<Ready>` instance.
  Splitting ownership across two instances (Option B, D) reintroduces aliasing.
  A single instance with a call-site CS selector (Option A) preserves the
  invariant.

- **Per-CS segment registers are disjoint but controller-scoped**: CS0 and CS1
  each have separate segment registers, but both are within the same MMIO
  block. The controller programs them both at `init()` and restores per-CS
  control words after each user-mode transaction. This is naturally expressed
  as per-CS state held inside a single controller instance — not split across
  two independent owners.

- **Capacity validation must be per-CS, not total**: the `SpiNorFlash` device
  facade's geometry parameter is validated against the *selected* CS's
  `FlashConfig`, not the sum of both CSes. A caller targeting CS0 with a
  1 MB config on a controller where CS0=2 MB and CS1=2 MB (total=4 MB) should
  get `InvalidCapacity`, not success. This requires the controller to retain
  per-CS config and check against the selected CS at facade construction time.

- **QEMU has one flash model on CS0 only**: the `ast1030-evb` QEMU machine
  attaches a `w25q80bl` to FMC CS0; CS1 has no device. Tests targeting CS1
  verify transport completeness (no hardware fault, CS line toggled) but do
  not assert read data. This is consistent with Option A — transport is CS
  line selection, not device presence.

- **aspeed-rust precedent**: aspeed-rust uses a flat array indexed by
  `master_idx` (not per-CS). Option A replaces the raw index with a typed
  enum while retaining the same array-dispatch pattern, making the port
  semantically faithful without importing C-style integer indexing.

## 5. Decision

**Option A** — `ChipSelect` enum at each call site; `cs0`/`cs1`
`Option<FlashConfig>` fields in `SmcConfig`.

Options B and D reintroduce MMIO aliasing, violating the single-owner
invariant. Option C loses the type-safety benefit for no architectural gain.
Option A is the only design that preserves single ownership, supports per-CS
config validation, and keeps the call surface auditable.

## 6. Consequences And Follow-ons

- All transport and facade APIs carry an explicit `ChipSelect` argument. In
  the common single-CS case this is `ChipSelect::Cs0`, which is cheap and
  clear.
- Adding a CS1 flash device to a board is a single change in the board
  crate (populate `cs1` with the appropriate `FlashConfig`); no controller
  or backend code changes.
- `SmcError::InvalidChipSelect` provides a clear runtime error if a caller
  requests CS1 on a CS0-only controller. A future decision could harden this
  to a compile-time error by encoding CS availability as a type parameter:
  `Smc<Ready, SingleCs>` vs `Smc<Ready, DualCs>`, where `transceive_user`
  accepts `ChipSelect::Cs1` only on `Smc<Ready, DualCs>`. This is Option D
  from §3. The tradeoff is that `init()` must return a different type
  depending on whether `config.cs1` was `Some` or `None` — either via two
  separate constructors or an enum-typed return — and every generic caller
  would need to be parameterized over the CS-count type. The runtime check is
  preferred until a concrete bug or audit finding justifies the added
  complexity.
- DMA is controller-scoped in this generation. If a future board requires
  independent DMA policies per CS, that is a follow-on decision.
