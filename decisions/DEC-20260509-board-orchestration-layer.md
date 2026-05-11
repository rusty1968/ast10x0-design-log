# Decision Record: Board crate as hardware orchestration layer

- Decision ID: DEC-20260509-board-orchestration-layer
- Status: accepted (retrospective — documents existing pattern; some sub-points aspirational, see §6)
- Owner: platform architecture
- Date: 2026-05-09
- Reviewers: firmware, platform, security
- Related decisions: DEC-20260509-peripheral-ownership-typestate, DEC-20260509-lifetime-bounded-device-facade, DEC-20260509-unsafe-consolidation-register-backend

## 1. Decision Question

How should board-level hardware orchestration be structured for AST10x0, given that production flash drivers run as separate server processes (with no MMIO access to SCU/SPIPF after handoff) while tests and the kernel setup phase need to coordinate the same hardware end-to-end in a single process?

## 2. Context And Constraints

- Technical context: SCU owns mux/routing registers (SCU0F0); SPIPF owns SPI monitor enforcement registers; SMC owns flash data-path controllers. A spimonitor's `set_mux` operation needs both SCU and SPIPF — the prior arrangement (monitor owning only SPIPF) had `set_mux` returning `HardwareError` because it could not reach SCU.
- Two distinct lifecycles share the same hardware:
  - **Setup phase** (kernel `main` or test process): mux + SPIPF policy + SPIPF lock are programmed exactly once. The SPIPF lock is irreversible by silicon — see DEC-20260503-spi-monitor-control-plane.
  - **Steady state** (per-process flash server): each server owns the SMC controller it drives, and must run *without* MMIO access to SCU/SPIPF to preserve per-process isolation.
- Security constraints: must not duplicate register-bit logic across crates (audit hazard); the SPIPF lock policy must be set in exactly one place.
- Reliability constraints: no allocation; single-threaded per process; ownership invariants from DEC-20260509-peripheral-ownership-typestate apply (single instance per hardware block).
- Performance constraints: zero overhead — orchestrators reduce to direct peripheral-helper calls.
- Delivery constraints: pattern must scale — additional orchestrators (clock, reset, etc.) should drop in without restructuring.
- Non-goals: providing a runtime that owns *every* peripheral (SMC controllers stay in their owning backend processes); generic-board portability beyond AST1060 in the current iteration.

## 3. Options Considered

### Option A: Board owns register blocks, lends to lifetime-bounded orchestrators, composes peripheral helpers (chosen)

- Summary: a `board` crate provides `Ast1060Board` owning `ScuRegisters` and `[SpiMonitorRegisters; 4]` (and future register blocks); methods like `board.monitor()` return `Ast1060Monitor<'a>` borrowing `&'a mut` of what it needs. Orchestrators delegate to peripheral helpers (`scu.set_spim_ext_mux(...)`) — never reimplement register logic. Controllers (`FmcReady`/`SpiReady`) are *not* owned by the board; they live in per-process backends. The SPIPF lock + the `_pre_wired` backend constructor family form the irreversible handoff seam between the setup-phase board and the steady-state backends.
- Benefits: solves the cross-peripheral-access problem (`set_mux` works because the orchestrator holds both SCU and SPIPF borrows); composition keeps register logic single-sourced; lifetime parameter prevents an orchestrator from outliving its borrows; per-process backends remain isolated because the handoff retires SCU/SPIPF access.
- Costs: lifetime parameters propagate to anything that stores an orchestrator; composition discipline is enforced by review, not by the type system; the board crate ends up with a small but real surface (descriptors + wiring + orchestrators + lock helpers).
- Main risks: a future orchestrator could be added that bypasses peripheral helpers and writes registers directly (covered in §7).
- Reversibility: low — code that takes `Ast1060Monitor<'_>` would need rewriting; the per-process / pre-wired seam is baked into backend constructors.

### Option B: No board crate; each peripheral owns whatever pieces of SCU/etc. it needs

- Summary: spimonitor pulls in SCU register fragments directly; smc pulls its own; etc.
- Benefits: no orchestration layer to maintain.
- Costs: SCU0F0 manipulation gets reimplemented in multiple crates; bit-level changes require multi-crate coordination; the original `set_mux` problem (monitor cannot reach SCU) returns.
- Main risks: silent drift between duplicate implementations; auditing becomes harder.
- Reversibility: high.

### Option C: Singleton / global accessor (`BOARD.lock().monitor()`)

- Summary: one global `Mutex<Ast1060Board>` (or `OnceCell`) accessed from anywhere.
- Benefits: no lifetime parameters; orchestrators reachable without passing `&mut board` around.
- Costs: introduces synchronization primitives where none are needed (single-threaded); makes hardware ownership non-local; defeats the per-process isolation guarantee that backends rely on.
- Main risks: erodes invariants from DEC-20260509-peripheral-ownership-typestate; tests sharing a global board cannot easily reset between phases.
- Reversibility: medium.

### Option D: Board owns initialized controllers too (`FmcReady` inside `Ast1060Board`)

- Summary: extend the board model down into the controller layer — board owns `FmcReady`/`SpiReady` alongside `ScuRegisters`.
- Benefits: single ownership tree for all hardware.
- Costs: cannot ship the board across the per-process boundary; flash backend would need to receive the controller from the board across IPC, which is not how the current process model works; the SPIPF lock handoff is awkward to express.
- Main risks: forces production flash servers to either run in the kernel process or take an architectural detour; breaks the per-process isolation that the `_pre_wired` backend variants are designed to preserve.
- Reversibility: low once adopted.

## 4. Evidence Summary

Source pattern proposed in [archive/BOARD_REFACTOR.md](../archive/BOARD_REFACTOR.md) and now realized in the openprot repo (`smc-peripheral` branch):

| Source | Supports | Confidence |
|---|---|---|
| `target/ast10x0/board/src/lib.rs` (`Ast1060Board` owns `ScuRegisters` + `[SpiMonitorRegisters; 4]`; `board.monitor()` lends `Ast1060Monitor<'_>`) | Board owns register blocks, hands out lifetime-bounded orchestrators | high |
| `target/ast10x0/board/src/monitor.rs` (`Ast1060Monitor<'a>` holds `&'a mut ScuRegisters` + `&'a mut [SpiMonitorRegisters; 4]`; `set_mux` calls `self.scu.set_spim_ext_mux(...)`) | Composition over duplication: orchestrators delegate to peripheral helpers | high |
| `target/ast10x0/backend/flash/src/lib.rs` (`Ast10x0FlashBackend` owns `FmcReady`/`SpiReady` + `LockedSpiMonitor` witness; uses `ast10x0_board` only for descriptors and `apply_spim_wiring`, not for `Ast1060Board`) | Controllers live in per-process backends, not on the board | high |
| `target/ast10x0/backend/flash/src/lib.rs:232-268` (`new_*_pre_wired` constructor family with the comment about preserving per-process isolation) | The SPIPF lock + pre-wired constructors form the setup-to-steady-state handoff seam | high |
| `target/ast10x0/tests/spim/test_boot_uc/target.rs` (`let mut board = unsafe { Ast1060Board::init() }; let mut monitor = board.monitor()`) | Pattern is exercised end-to-end | high |
| `target/ast10x0/board/` does not exist as `board_descriptors/` (renamed) | Migration step from BOARD_REFACTOR.md is complete | high |

## 5. Scorecard Result

Not applicable — retrospective.

## 6. Decision

- Chosen option: Option A.
- Concretely:
  1. **Board owns register blocks**, not controllers. `Ast1060Board` holds `ScuRegisters` and `[SpiMonitorRegisters; 4]`; controllers (`FmcReady`/`SpiReady`) and lock witnesses (`LockedSpiMonitor`) live in per-process backends.
  2. **Orchestrators borrow from the board** with explicit lifetime parameters (`Ast1060Monitor<'a>`), constructed via `board.method()` accessors.
  3. **Composition over duplication** is the binding rule: orchestrators delegate to peripheral helpers (`scu.set_spim_ext_mux`, `regs.read_ctrl`, etc.) — they never write registers directly. Type conversions between layer-specific enums (e.g., `MonitorInstance` ↔ `SpiMonitorInstance`) live in the orchestrator.
  4. **Setup-phase / steady-state handoff** is via the irreversible SPIPF lock + the `_pre_wired` backend constructor family. After handoff, server processes hold no SCU/SPIPF access; the board (or equivalent setup code) is no longer in the picture.
- Aspirational sub-points carried forward from BOARD_REFACTOR.md, not yet realized:
  - **"Other orchestrators follow same pattern."** Only `board.monitor()` exists today; `board.clock()`, `board.reset()` etc. are intended but not implemented. New orchestrators must follow the same shape (return `Foo<'_>` borrowing from the board).
  - **"Multi-board generic design."** `Ast1060Board` is concrete. The trait-bounded form (`Board<UART, SPIPF, SCU>`) is on the table for the future but not in code.
- Rationale: solves the original cross-peripheral-access problem (monitor can finally reach SCU) without sacrificing the per-process isolation that production flash servers depend on. The board crate becomes the *setup-phase* runtime; per-process backends become the *steady-state* runtime; the SPIPF lock is the contract between them. Composition discipline keeps register logic single-sourced.
- Why now: the orchestration layer landed; the per-process / pre-wired seam landed; locking the rule down keeps future orchestrators consistent with both the typestate ownership model and the per-process isolation model.

## 7. Risks And Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
|---|---|---|---|---|
| A future orchestrator writes SCU/SPIPF registers directly instead of delegating to peripheral helpers | med | med | code-review rule: orchestrators (`board/src/*.rs`) must not contain `regs.write_*` for register fields owned by another peripheral; require justification for any direct register access | platform architecture |
| Single-process test that combines `Ast1060Board::init()` with a non-pre-wired backend will alias SCU (both call `ScuRegisters::new_global()`) | med | med | tests that already construct an `Ast1060Board` must use `_pre_wired` backend constructors and program SPIM wiring through the board; reviewer responsibility | firmware |
| `Ast10x0FlashBackend::new_with_descriptor` reaches around the board to grab SCU itself (line 206), violating the "single instance" contract if a board exists in the same process | med | high | the non-pre-wired path is intended for production server processes that own all the hardware they touch (no board present); document explicitly that `_pre_wired` is the only sanctioned path when a board lives in the same process | platform architecture |
| Aspirational sub-points (other orchestrators, generic multi-board) drift if added later under pressure without re-reading this ADR | med | low | reference this DEC in the orchestrator-shaped CLAUDE.md note; require new orchestrators to cite it in PR descriptions | platform architecture |
| Lifetime parameter on orchestrator types pushes contributors toward unsafe shortcuts (e.g., storing an orchestrator in a `'static` somewhere) | low | high | follow the same rule as DEC-20260509-lifetime-bounded-device-facade — no `'static` storage of orchestrator-shaped types; review will flag | platform architecture |
| The "board ends here, backends own from here" handoff is a convention enforced by SPIPF lock semantics, not by the type system | low | med | the SPIPF lock is silicon-irreversible (per DEC-20260503-spi-monitor-control-plane), which gives the convention a hardware-level backstop; pre-wired backend variants are the only way to use a controller without re-asserting wiring | firmware + security |

## 8. Validation Plan

- Existing test: `target/ast10x0/tests/spim/test_boot_uc/target.rs` runs the full hold → configure-policy → release → verify sequence through `Ast1060Board::init()` + `board.monitor()`.
- Composition audit: grep `target/ast10x0/board/src/` for direct register writes that bypass peripheral helpers — expected to find only orchestrator-internal type conversions, never `regs.write_*` for fields owned by another peripheral's helper.
- Handoff audit: grep `target/ast10x0/backend/` for use of `Ast1060Board` — expected to be zero. Backends use `Ast10x0BoardDescriptor`, `apply_spim_wiring`, and `SpimWiringError` from `ast10x0_board`, not the runtime owner.
- Required for new orchestrators: take `&mut self` on the board, return `Orchestrator<'_>` borrowing the register blocks the orchestrator needs; delegate to peripheral helpers; do not introduce runtime sync.

## 9. Revisit Trigger

- Revisit date: 2026-11-09
- Early revisit signals:
  - More than one backend needs to construct its own `ScuRegisters::new_global()` in the same process — suggests the pre-wired seam is in the wrong place
  - Orchestrator count grows past ~3–4 and the composition-over-duplication discipline starts to slip
  - Multi-board support is requested in earnest — re-evaluate the generic trait-bound design from BOARD_REFACTOR.md §"Architecture Decisions" 3
  - Per-process isolation requirement changes (e.g., kernel-resident flash drivers) — board could legitimately own controllers, collapsing the two-tree split
  - A new peripheral arrives that genuinely needs cross-orchestrator access and breaks the "one orchestrator per concern" assumption

## 10. Copilot Audit Prompt

"Review this decision for unsupported claims, stale evidence, and missing failure modes. Provide concrete gaps with file links and suggested fixes."
