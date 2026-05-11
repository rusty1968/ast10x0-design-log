# Decision Record: Lifetime-bounded device facades over controllers

- Decision ID: DEC-20260509-lifetime-bounded-device-facade
- Status: accepted (retrospective — documents existing pattern)
- Owner: platform architecture
- Date: 2026-05-09
- Reviewers: firmware, platform
- Related decisions: DEC-20260509-peripheral-ownership-typestate

## 1. Decision Question

How should higher-level device abstractions (e.g., `SpiNorFlash`) reference their underlying controllers: by owning the controller, by holding a `'static` raw pointer, or by holding a lifetime-bounded mutable borrow?

## 2. Context And Constraints

- Technical context: a single SMC controller can drive different flash devices through different chip selects and transports (FMC vs SPI). The device facade encodes flash-level concerns (capacity, addressing policy, command profile); the controller encodes bus-level concerns. They have different lifetimes and shouldn't be fused.
- Security constraints: facade must not outlive its controller (use-after-free on hardware would be a real fault, not a memory bug).
- Reliability constraints: must work with the type-state controller — the facade has to be usable while the controller is in a `Ready`-like state and not interfere with state transitions.
- Performance constraints: zero-cost — borrow is just a pointer; no allocation.
- Delivery constraints: must compose with `DEC-20260509-peripheral-ownership-typestate` (no Rc/Arc).
- Non-goals: long-lived facades stored in globals; cross-task facade sharing.

## 3. Options Considered

### Option A: `Facade<'a>` borrows the controller via lifetime parameter (chosen)

- Summary: `SpiNorFlash<'a>` holds an enum (`FlashBackend<'a>`) wrapping `&'a mut FmcReady` or `&'a mut SpiReady`. Constructors take the controller borrow; the borrow checker enforces controller-outlives-facade.
- Benefits: compile-time use-after-free prevention; controller can be reused (or reconfigured) once the facade is dropped; one controller can be lent to different facades sequentially; works directly with the type-state pattern (the borrow can require `Ready`).
- Costs: lifetime parameter propagates into types that store the facade; cannot store facade and controller in the same struct without self-referential workarounds.
- Main risks: callers may try to keep facades around longer than the borrow allows, leading to friction; lifetime annotations spread through call sites.
- Reversibility: medium — switching to owned controllers later is mechanical but touches every storage site.

### Option B: Facade owns the controller (move it in)

- Summary: `SpiNorFlash` takes the controller by value; gives it back via a `release()` method.
- Benefits: no lifetime parameters; facade can be stored anywhere.
- Costs: cannot use the controller for anything else while the facade exists; hard to support multiple facades over different chip selects; `release()` adds an explicit step every caller must remember.
- Main risks: precludes scenarios where the bus must be re-borrowed by a different abstraction (e.g., direct user-mode SPI for a non-flash device).
- Reversibility: high.

### Option C: Facade holds a `'static` raw pointer or copy of the backend

- Summary: bypass the borrow checker; trust the programmer.
- Benefits: facade is `'static`; trivially storable.
- Costs: throws away the use-after-free guarantee; introduces aliasing that the rest of the codebase has been carefully avoiding (see DEC-20260509-peripheral-ownership-typestate).
- Main risks: inconsistent with the project-wide ownership model.
- Reversibility: low — would erode the invariants other decisions depend on.

## 4. Evidence Summary

Source pattern documented in [archive/PERIPHERAL_OWNERSHIP_ANALYSIS.md](../archive/PERIPHERAL_OWNERSHIP_ANALYSIS.md), sections 2.4 and 7.2. Concrete instances in the openprot repo (`smc-peripheral` branch):

| Source | Supports | Confidence |
|---|---|---|
| `target/ast10x0/peripherals/smc/device/flash.rs` (`SpiNorFlash<'a>` with `FlashBackend<'a>`) | Lifetime-bounded facade pattern | high |
| `target/ast10x0/peripherals/smc/device/flash.rs::from_fmc` (`fmc: &'a mut FmcReady`) | Constructor encodes borrow + `Ready` requirement | high |
| `target/ast10x0/tests/smc/target_device.rs` (creates facade after controller init, uses it, drops it) | End-to-end pattern works | high |

## 5. Scorecard Result

Not applicable — retrospective.

## 6. Decision

- Chosen option: Option A. Device facades take the controller by mutable borrow with an explicit lifetime, requiring the type-state to be at the appropriate stage (`Ready`).
- Rationale: composes cleanly with the type-state ownership model — the facade depends on a specific lifecycle stage at the type level, and use-after-free is structurally impossible. The cost (lifetime annotations propagating through callers) is bounded; the benefit (eliminating an entire class of hardware-aliasing bugs at compile time) is permanent.
- Why now: pattern is already in place for `SpiNorFlash`; locking it in keeps future device abstractions (other flash families, devices behind spimonitor's transparent path) consistent.

## 7. Risks And Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
|---|---|---|---|---|
| Future need to store a facade in a board-level struct collides with lifetime parameter | med | med | board-level storage holds the controller; facades are constructed at use-site and dropped after the operation | firmware |
| Two facades over the same controller (e.g., different chip selects) need to coexist | med | med | revisit by allowing facades to borrow different sub-resources of the controller (per-CS state) rather than the whole controller | platform architecture |
| Lifetime gymnastics push contributors toward unsafe shortcuts | low | high | document the pattern in CLAUDE.md; review any `'static` storage of facade-shaped types | platform architecture |
| Async/RTOS task adoption later requires owned facades or `Pin`-style storage | low | med | acceptable to revisit at that point; current single-threaded model doesn't pay this cost | platform architecture |

## 8. Validation Plan

- Existing test: `target/ast10x0/tests/smc/target_device.rs` — `SpiNorFlash::from_fmc(&mut fmc, FLASH_CFG)?` then `flash.read(...)` then implicit drop.
- Compile-time check: an attempt to drop or move the controller while the facade exists must fail to compile.
- Required for new device facades: borrow the underlying controller with a lifetime parameter; do not introduce `'static` shortcuts.

## 9. Revisit Trigger

- Revisit date: 2026-11-09
- Early revisit signals: introduction of an async runtime or RTOS that needs facades crossing await points; need for two simultaneous facades against the same controller and per-CS borrowing isn't workable; a board-storage requirement that genuinely cannot be solved with construct-on-demand.

## 10. Copilot Audit Prompt

"Review this decision for unsupported claims, stale evidence, and missing failure modes. Provide concrete gaps with file links and suggested fixes."
