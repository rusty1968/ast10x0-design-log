# Decision Record: Concentrate `unsafe` MMIO in register backends

- Decision ID: DEC-20260509-unsafe-consolidation-register-backend
- Status: accepted (retrospective — documents existing pattern)
- Owner: platform architecture
- Date: 2026-05-09
- Reviewers: firmware, platform
- Related decisions: DEC-20260509-peripheral-ownership-typestate

## 1. Decision Question

Where should `unsafe` MMIO access live: scattered through controller/driver code at each register touch, or concentrated behind dedicated register-backend types whose internals are the single audit perimeter?

## 2. Context And Constraints

- Technical context: AST10x0 peripherals are accessed via raw pointer dereferences into hardware register blocks. The PAC exposes typed register block structs but the pointer-to-block step itself is `unsafe`.
- Security constraints: every `unsafe` block needs a written safety contract; concentrating them makes audit tractable.
- Reliability constraints: duplicating raw-pointer construction in each operation increases the chance one of them gets the contract wrong.
- Performance constraints: zero overhead — backend wrappers must be `#[inline]` and reduce to the same code as direct pointer use.
- Delivery constraints: pattern must scale — every new peripheral needs a backend.
- Non-goals: a generic HAL crate; replacing the PAC.

## 3. Options Considered

### Option A: Backend struct with single `regs()` audit point (chosen)

- Summary: each peripheral has a backend type (e.g., `FmcRegisterBackend`, `SpiRegisterBackend`, `SpiMonitorRegisters`) holding a raw `*const RegisterBlock`. An `unsafe` constructor records the safety contract once. A private `regs()` method returns `&RegisterBlock`; this is the only place `unsafe { &*self.base }` appears. All typed register operations go through this method.
- Benefits: one place to audit per peripheral; safety contract is documented at the constructor; controller code is `unsafe`-free; backends are swappable (FMC vs SPI sharing the same controller logic).
- Costs: an extra struct per peripheral; volatile reads of register fields outside the typed PAC API need their own helpers (still inside the backend) to honor the same audit perimeter.
- Main risks: discipline — ad-hoc volatile reads can drift outside the backend if not policed.
- Reversibility: medium.

### Option B: Inline `unsafe` at each register access

- Summary: every operation builds its own `&*ptr` or `read_volatile(ptr)` directly.
- Benefits: no abstraction layer.
- Costs: many small `unsafe` blocks, each duplicating the same safety contract; auditing requires reading every site.
- Main risks: contract drift across sites; easy to miss one when invariants change.
- Reversibility: high.

### Option C: Adopt or build a generic HAL with `volatile-register`/`vcell` everywhere

- Summary: typed cells encapsulate every field; no raw pointers visible to driver code.
- Benefits: stronger per-field safety story.
- Costs: requires PAC redesign; doesn't help with operations that have to bypass typed access (raw offset reads of volatile log indices, byte/word streaming through the user-mode SPI window).
- Main risks: large rewrite for marginal additional safety beyond Option A.
- Reversibility: low once adopted.

## 4. Evidence Summary

Pattern was originally surveyed in a peripheral-ownership planning doc (since removed from `archive/`; recoverable from git history). Concrete instances in the openprot repo (`smc-peripheral` branch):

| Source | Supports | Confidence |
|---|---|---|
| `target/ast10x0/peripherals/smc/fmc_backend.rs` (`FmcRegisterBackend::new` `unsafe`, `regs()` audit point) | Single-point unsafe consolidation | high |
| `target/ast10x0/peripherals/smc/controller.rs` (controller body uses `self.regs.write_cs_ctrl(...)`, no inline `unsafe`) | Driver code stays safe by construction | high |
| `target/ast10x0/peripherals/spimonitor/registers.rs` (`read_log_idx_reg`, `log_ram_base_addr` use `read_volatile` inside the backend) | Volatile/raw-offset reads stay inside the backend perimeter | medium — these helpers bypass `regs()`; the pattern allows it but discipline is on the author |
| `target/ast10x0/peripherals/smc/controller.rs::transceive_user` (raw MMIO window writes via `spi_write_data`/`spi_read_data`) | Where typed access does not apply, `unsafe` is still localized to a documented helper | high |

## 5. Scorecard Result

Not applicable — retrospective.

## 6. Decision

- Chosen option: Option A. Each peripheral has a backend type; the backend's constructor carries the safety contract; `regs()` is the typed audit point; volatile/raw-offset helpers live inside the backend.
- Rationale: keeps controller code free of `unsafe` and free of register-pointer arithmetic; reviewers reading a controller can take backend correctness as a precondition and focus on protocol logic.
- Why now: pattern is already implemented for SMC and spimonitor backends; documenting it locks in the audit perimeter so new peripherals don't grow inline `unsafe`.

## 7. Risks And Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
|---|---|---|---|---|
| Volatile reads sneak into controller code instead of staying in the backend | med | med | code review checklist: any `read_volatile`/`write_volatile` outside a `*_backend.rs` or `registers.rs` file requires a written justification | firmware |
| Backend's "single instance per hardware block" precondition violated by careless caller | low | high | board-level init is the only sanctioned construction site; constructors are `pub unsafe`, callers must justify | firmware |
| User-mode SPI helpers (`spi_read_data`/`spi_write_data`) live outside the backend file | low | low | acceptable — they target the AHB flash window, not the register block; document the split in their safety comments | firmware |
| New PAC version changes register block layout without backend update | low | med | backend constructor and `regs()` are the only `unsafe { &*ptr }` sites — a PAC bump only needs review of those files | platform architecture |

## 8. Validation Plan

- Audit: grep `target/ast10x0/peripherals/` for `unsafe` blocks; expected locations are `*_backend.rs`, `registers.rs`, the `transceive_user` user-mode SPI helpers, and constructor `unsafe fn`s on controller types. Anything else is a finding.
- Existing tests: `target/ast10x0/tests/smc/target.rs` exercises the SMC backend path end-to-end; `target/ast10x0/tests/spim/test_boot_uc/target.rs` exercises the spimonitor register helpers.

## 9. Revisit Trigger

- Revisit date: 2026-11-09
- Early revisit signals: a peripheral arrives where the typed PAC is unusable and inline `unsafe` becomes tempting; PAC adopts `volatile-register`/`vcell` and the backend wrapper becomes redundant; security review flags the backend perimeter as too coarse for a specific peripheral.

## 10. Copilot Audit Prompt

"Review this decision for unsupported claims, stale evidence, and missing failure modes. Provide concrete gaps with file links and suggested fixes."
