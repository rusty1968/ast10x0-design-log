# Decision Record: Model SMC controller topology as `SmcTopology` enum

- Decision ID: DEC-20260510-smc-topology-modeling
- Status: accepted (retrospective for the data model + board mapping; controller-side behavior gating aspirational, see §6)
- Owner: platform architecture
- Date: 2026-05-10
- Reviewers: firmware, platform
- Related decisions: DEC-20260509-board-orchestration-layer, DEC-20260509-peripheral-ownership-typestate, DEC-20260509-unsafe-consolidation-register-backend

## 1. Decision Question

How should AST10x0 SMC controller topology (the aspeed-rust `ctrl_type` + `master_idx` pair) be modeled in code, and which layer is responsible for the data, the controller-to-topology mapping, and the role-dependent behavior?

## 2. Context And Constraints

- Technical context: aspeed-rust's `SpiConfig` carries two fields that gate runtime behavior in its SPI controller — `ctrl_type` (`BootSpi`/`HostSpi`/`NormalSpi`) and `master_idx` (0 for FMC/SPI1, 2 for SPI2 in current usage). These influence decode-window sizing/pre-init, calibration skip on CS1 for nonzero `master_idx`, some HostSpi-specific SPI06C/SPI074 register programming, and CS routing under nonzero `master_idx`.
- The AST10x0 SMC stack already models controller identity (`SmcController::{Fmc, Spi1, Spi2}`), CSes, DMA, IRQs — but did not carry the topology, and the controller layer cannot make role-dependent decisions without it.
- Composition with prior decisions:
  - DEC-20260509-board-orchestration-layer says the board crate is the source of truth for board-specific declarations.
  - DEC-20260509-peripheral-ownership-typestate says peripheral lifecycle stages are encoded as types so invalid sequences are unrepresentable. The same principle — make important state machine-checkable rather than inference-based — extends to controller role here.
  - DEC-20260509-unsafe-consolidation-register-backend says backends stay register-transport only.
- Constraints: thin SPI wrapper (`spi.rs`) must not become a policy engine; register backends must not own role decisions; aspeed-rust parity should be semantic, not syntactic.
- Non-goals: redesigning the public SPI facade; rewriting aspeed-rust's controller layer line-for-line; introducing a fourth role variant beyond Boot/Host/Normal until hardware proves it's needed.

## 3. Options Considered

### Option A: Discriminated enum `SmcTopology { BootSpi { master_idx }, HostSpi { master_idx }, NormalSpi { master_idx } }` (chosen)

- Summary: topology is a single enum field on `SmcConfig`; each variant carries its `master_idx`. Board crate populates it from the descriptor. Controller layer gates behavior with `match`/`matches!(topology, SmcTopology::HostSpi { .. })`. SPI wrapper stays thin; backends stay register-transport.
- Benefits: ambiguous states are unrepresentable (no "FMC-with-HostSpi" or "SPI2-pretending-to-be-BootSpi" unless explicitly constructed); role and `master_idx` are coupled in one field rather than scattered; pattern matching is the natural call-site idiom.
- Costs: introducing a new enum and threading it through `SmcConfig`; helper methods (`master_idx()`, `is_host_spi()`, etc.) tempt scope creep.
- Main risks: controller-side gating on `topology` is not yet wired (see §6); the data model exists but the behavior it's meant to gate is still controller-id-based or hardcoded.
- Reversibility: low — changing the enum shape later would touch `SmcConfig` construction sites and every consumer pattern-match.

### Option B: Carry only `master_idx: u8`; let controller logic infer role from `controller_id + master_idx`

- Summary: minimal data; the role is implicit.
- Benefits: smaller `SmcConfig`; no new type.
- Costs: every consumer needs the role-derivation rule; ambiguous states like "FMC with `master_idx == 2`" become representable; aspeed-rust's `HostSpi` semantics are tied to *role*, not just `master_idx`, so the inference rule has to live somewhere — likely scattered.
- Main risks: role-derivation drift across consumers; subtle incorrect behavior when `master_idx`/role pairings change.
- Reversibility: medium.

### Option C: Boolean flags (`is_boot_spi`, `is_host_spi`, `host_id`)

- Summary: separate flags rather than a discriminated union.
- Benefits: no enum.
- Costs: ambiguous states are representable (two flags set true; neither set; flags inconsistent with `host_id`); validation logic must be added at construction time.
- Main risks: invariants not enforceable at the type level; the borrow checker provides no help.
- Reversibility: medium.

### Option D: Put role-dependent behavior in the SPI wrapper (`spi.rs`)

- Summary: keep `controller.rs` "controller-id agnostic"; let `spi.rs` carry the role-specific paths.
- Benefits: preserves the appearance of a "shared" controller.
- Costs: bifurcates the FMC vs SPI handling — FMC's analogous logic already lives in the controller; SPI's would be in the wrapper. Two layouts for the same kind of decision.
- Main risks: violates DEC-20260509-board-orchestration-layer composition discipline ("delegate, don't reimplement"); breaks the wrapper's thin-by-design role.
- Reversibility: medium.

## 4. Evidence Summary

Pattern was originally surveyed in an SPI topology planning doc (since removed from `archive/`; recoverable from git history). Concrete state in the openprot repo (`smc-peripheral` branch):

| Source | Supports | Confidence |
|---|---|---|
| `target/ast10x0/peripherals/smc/types.rs` (`SmcTopology` enum + `SmcConfig.topology` field) | Data model is in code | high |
| `target/ast10x0/board/src/lib.rs` (`Ast10x0BoardDescriptor::smc_config()`: matches `controller` to topology) | Board crate *encodes* the mapping rule; helper is correct in isolation | high |
| `Ast10x0BoardDescriptor::smc_config()` has no callers in the openprot tree (confirmed by repo-wide grep) | The mapping helper is dead code in production today; the board crate is not yet a *de facto* single source of truth | high |
| `target/ast10x0/backend/flash/src/lib.rs` (`build_smc_controller` constructs `SmcConfig` from scratch with `topology: SmcTopology::BootSpi { master_idx: 0 }` and a TODO comment to refine per controller) | Backend hardcodes topology regardless of controller — the descriptor's correct value is built but discarded | high |
| `target/ast10x0/peripherals/smc/controller.rs` does not pattern-match or branch on `topology` | Controller-side gating on the field is not yet wired | high |
| aspeed-rust `src/spi/types.rs` (`CtrlType`, `master_idx` in `SpiConfig`); `src/spi/spicontroller.rs` (uses both fields to gate decode-range and calibration); `src/spi/spitest.rs` (mappings: FMC→BootSpi/0, SPI1→HostSpi/0, SPI2→NormalSpi/2) | Source of the model and the mapping | high |

## 5. Scorecard Result

Not applicable — retrospective.

## 6. Decision

- Chosen option: Option A.
- Concretely:
  1. **Topology is `SmcTopology`**, carried as a field on `SmcConfig`. Variants: `BootSpi { master_idx: u8 }`, `HostSpi { master_idx: u8 }`, `NormalSpi { master_idx: u8 }`.
  2. **Board crate is the intended single source of truth** for the controller→topology mapping. `Ast10x0BoardDescriptor::smc_config()` encodes the mapping. *Today this helper has zero callers — the production flash backend assembles `SmcConfig` field-by-field and bypasses it entirely.* The plumbing fix in §6 is to route production construction through `smc_config()` (or otherwise consult the descriptor's topology), so the helper becomes the de facto source of truth and not just the on-paper one.
  3. **Mapping table** (treat as code-generation truth for board presets):

     | Controller | Topology | master_idx |
     |---|---|---|
     | `Fmc` | `BootSpi` | `0` |
     | `Spi1` | `HostSpi` | `0` |
     | `Spi2` | `NormalSpi` | `2` |

  4. **Controller layer (`controller.rs`) is the only place that consumes topology to gate behavior.** SPI wrapper (`spi.rs`) stays a thin construction/lifecycle layer. Register backends stay register-transport only (per DEC-20260509-unsafe-consolidation-register-backend).
  5. **Aspeed-rust parity is "port semantics, not syntax"**: only the behaviors that demonstrably matter for AST10x0 hardware get ported, gated via `matches!(topology, ...)` rather than `match controller_id`.
- Aspirational sub-points (not yet in code) — two distinct gaps:
  - **Plumbing gap (backend).** `build_smc_controller` in the flash backend constructs `SmcConfig` field-by-field and hardcodes `topology: BootSpi { master_idx: 0 }`. It does not call `Ast10x0BoardDescriptor::smc_config()` — which is why that helper currently has zero callers. The fix is local: route construction through `smc_config()` (or read `descriptor.smc_config().topology` and use it). The TODO comment in `build_smc_controller` is the tracking marker.
  - **Behavior gap (controller).** `controller.rs` does not yet pattern-match on `topology` to gate role-dependent behavior. Three concrete behaviors carried over from aspeed-rust are worth porting once the plumbing is fixed:
    - Decode-range sizing / pre-init for `master_idx != 0`.
    - Calibration skip for nonzero `master_idx` on CS1.
    - HostSpi-specific SPI06C/SPI074 control programming via `matches!(topology, SmcTopology::HostSpi { .. })` rather than controller-id heuristics.
- Rationale: a discriminated union is the only option that makes invalid topologies unrepresentable at the type level. The board-as-single-source-of-truth principle composes cleanly with DEC-20260509-board-orchestration-layer. The controller-layer gating rule keeps role logic in one place — extending the same composition-over-duplication discipline already established for SCU/SPIPF.

  **Why machine-checkable rather than inference-based.** "Inference-based" means the value isn't stored directly; consumers have to compute it from other fields. The rejected Option B is the inference-based shape: store `controller_id` and `master_idx`, let consumers derive the role:

  ```rust
  let role = match (config.controller_id, config.master_idx) {
      (SmcController::Fmc, 0)  => Role::BootSpi,
      (SmcController::Spi1, 0) => Role::HostSpi,
      (SmcController::Spi2, 2) => Role::NormalSpi,
      _ => /* what now? */,
  };
  ```

  The role exists only as the *output* of that derivation — it isn't in the data, it's a computation over the data. Three problems fall out:

  1. **The rule has to live somewhere.** If two consumers need the role, they either share a helper (better) or each implement their own version (drift risk).
  2. **Ambiguous inputs become representable.** What does `(SmcController::Fmc, master_idx: 2)` mean? Under inference, every consumer has to handle it — usually with a default, an error, or a panic. With the discriminated enum, it is unconstructable.
  3. **The intent gets lost at construction time.** The board crate *knows* the role when it builds the descriptor (`SPI1 → HostSpi`). Inference throws that knowledge away and forces every reader to reconstruct it.

  Pattern matching the discriminated enum has none of these properties: the role is stored as a tagged variant, no rule lives anywhere, no drift is possible, pattern matching is exhaustive (the compiler tells you when you've forgotten a variant), and invalid pairings literally do not exist as values.

  Same pattern at a different layer in DEC-20260509-peripheral-ownership-typestate: lifecycle stage is *not* a `state: SmcState` runtime field you have to inspect — it's `Smc<B, Ready>` vs `Smc<B, Uninitialized>`, where the type itself carries the information and inspection is unnecessary.
- Why now: the data model and the board crate's mapping have landed; locking down those rules now keeps the upcoming backend plumbing fix and controller-side gating honest, and prevents controller-id heuristics from creeping back in once role-dependent behavior starts landing.

## 7. Risks And Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
|---|---|---|---|---|
| Backend hardcodes `topology: BootSpi { master_idx: 0 }` and bypasses `Ast10x0BoardDescriptor::smc_config()` entirely (which is why that helper currently has zero callers); SPI1/SPI2 backends pass the wrong topology to the controller | high (already true today) | low while the controller does no gating; med once gating lands | local fix in `build_smc_controller`: route through `smc_config()` or read `descriptor.smc_config().topology`; the TODO comment in that function is the tracking marker | firmware |
| Controller never starts gating behavior on `topology`; the field stays decorative; SPI1/SPI2 controller behavior continues using whatever the controller-id-based code path does today | med | med | revisit signal in §9; the readiness checklist in §8 names the three concrete sites that need wiring | firmware |
| New role-dependent behavior gets added in `spi.rs` instead of `controller.rs` because the wrapper is "closer" | med | med | code-review rule: any `match`/`matches!` on `SmcTopology` outside `controller.rs` is a finding requiring justification | platform architecture |
| Helper methods grow before they're needed (`is_host_spi()`, `master_idx()`, `is_boot_spi()`) and accumulate API surface | low | low | only add helpers when a real call site needs them; `match` on the enum is the default | platform architecture |
| Mapping table baked into board presets diverges if a future board needs a non-default pairing (e.g., a board where SPI2 plays HostSpi) | low | med | descriptors are the only construction site; new boards add new descriptor constructors, not new mapping rules; the enum allows per-descriptor overrides without changing the controller layer | platform architecture |
| Aspeed-rust port becomes mechanical: a quirky aspeed-rust branch ported as-is into the controller without verifying AST10x0 actually needs it | med | low | "port semantics, not syntax" — each ported branch needs an explicit reason tied to AST10x0 hardware behavior, not "aspeed-rust does X" | firmware |
| Board/peripheral drift: board mapping and controller semantics evolve in separate PRs and the mapping diverges | low | med | board crate is the only `SmcConfig` construction site outside backend pre-wired paths; reviewer enforcement | platform architecture |

## 8. Validation Plan

- Compile check: `bazelisk build --config=virt_ast10x0 //target/ast10x0/peripherals:peripherals //target/ast10x0/board:ast10x0_board` — ensures the data model holds across the construction sites.
- Board mapping invariant: `Ast10x0BoardDescriptor::smc_config()` returns a topology consistent with `controller_id` per the table above; could become a small unit test on the descriptor module.
- Readiness checklist (required before declaring topology gating wired end-to-end):
  - backend constructor (`build_smc_controller`) sources `topology` from the descriptor instead of the hardcoded `BootSpi { master_idx: 0 }` — the plumbing gap from §6
  - decode-range sizing in controller logic uses `topology.master_idx()` (or pattern match) rather than `match controller_id`
  - calibration skip path for CS1 references `topology.master_idx() != 0` rather than `controller_id == Spi2`
  - SPI06C/SPI074 programming gated on `matches!(topology, SmcTopology::HostSpi { .. })`
- Behavior validation surface (once the readiness checklist clears): SPI1 host-topology init; SPI2 normal-topology init; CS1 behavior on SPI2 (`master_idx = 2`).

## 9. Revisit Trigger

- Revisit date: 2026-11-10
- Early revisit signals:
  - The §8 readiness checklist clears (backend sources topology from descriptor; controller gates on it) — promote status from mixed to fully retrospective and remove the aspirational caveat from §6
  - A new SoC variant or board config breaks the simple controller→topology mapping (e.g., dual-role controllers)
  - aspeed-rust drift surfaces semantics worth porting that don't fit the current three-variant enum
  - Backend's hardcoded `BootSpi { master_idx: 0 }` causes a real bug in production — the TODO has aged into a defect
  - Multiple boards need different topology mappings for the same controller — re-evaluate whether mapping should move out of `smc_config()` into per-descriptor overrides

## 10. Copilot Audit Prompt

"Review this decision for unsupported claims, stale evidence, and missing failure modes. Provide concrete gaps with file links and suggested fixes."
