# Decision Record: Peripheral ownership via type-state and exclusive ownership

- Decision ID: DEC-20260509-peripheral-ownership-typestate
- Status: accepted (retrospective — documents existing pattern)
- Owner: platform architecture
- Date: 2026-05-09
- Reviewers: firmware, platform
- Related decisions: DEC-20260509-unsafe-consolidation-register-backend, DEC-20260509-lifetime-bounded-device-facade

## 1. Decision Question

How should AST10x0 peripheral lifecycle and ownership be modeled: shared ownership (Rc/Arc), runtime-checked single ownership (RefCell, singleton-with-`take`), or compile-time enforced exclusive ownership with type-state?

## 2. Context And Constraints

- Technical context: bare-metal Cortex-M, single-threaded execution per peripheral. Hardware blocks (SMC/FMC, SPI monitor, UART) require strict init-before-use ordering; some have multi-stage lifecycles (e.g., spimonitor: Uninitialized → Configured → Locked, where Locked is irreversible).
- Security constraints: invalid API sequences (use-before-init, double-init) must be unreachable, not merely defended against.
- Reliability constraints: zero allocation; deterministic state machines.
- Performance constraints: zero runtime cost for ownership tracking.
- Delivery constraints: pattern must be uniform across peripherals so new drivers don't reinvent it.
- Non-goals: cross-task sharing of a peripheral handle; dynamic discovery of peripherals.

## 3. Options Considered

### Option A: Type-state with PhantomData + exclusive ownership (chosen)

- Summary: peripheral struct generic over a `Mode` marker (`Uninitialized`, `Ready`, `Configured`, `Locked`); `PhantomData<fn() -> Mode>` carries it at zero cost. Init consumes the previous state and returns the next. No Rc/Arc.
- Benefits: invalid sequences fail to compile; no runtime overhead; uniform across peripherals; pairs naturally with `&self` / `&mut self` distinction for read vs state-changing operations.
- Costs: more generic noise in signatures; new contributors must learn the pattern.
- Main risks: pattern is convention-enforced for new drivers; not a trait the compiler requires.
- Reversibility: low — code that depends on `Smc<B, Ready>` would all need rewriting.

### Option B: Shared ownership via Rc/Arc + runtime borrow checks

- Summary: handles freely cloned, hardware access guarded by RefCell or similar.
- Benefits: handles can flow anywhere in the program.
- Costs: runtime checks cost cycles and can panic late; requires `alloc`; defeats compile-time lifecycle modelling.
- Main risks: late-detected ownership violations in a single-threaded context that has no actual sharing need.
- Reversibility: medium.

### Option C: Singleton with runtime `take()` (cortex_m-style)

- Summary: one global instance per peripheral, retrieved once with a runtime `Option::take`.
- Benefits: prevents duplicate instances at runtime; no generics.
- Costs: lifecycle stages still expressed as runtime state; no compile-time use-before-init prevention; `take()` returns `Option`, pushing failure into runtime.
- Main risks: weaker invariants; relies on caller to never double-take.
- Reversibility: medium.

### Counterargument from the literature

Ginger Bill, *The Fatal Flaw of Ownership Semantics* (2020), argues that ownership-as-a-paradigm imposes a linear hierarchy ill-suited to graph-shaped data and recommends handles + generations instead — accepted here as out of scope, since AST10x0 peripherals are single-instance, single-threaded hardware resources rather than graph-shaped long-lived data. Revisit signals in §9 cover the conditions under which this critique would start to bite.

## 4. Evidence Summary

Source pattern documented in [archive/PERIPHERAL_OWNERSHIP_ANALYSIS.md](../archive/PERIPHERAL_OWNERSHIP_ANALYSIS.md), sections 1, 2, 4, 7.1, 7.3. Concrete instances in the openprot repo (`smc-peripheral` branch):

| Source | Supports | Confidence |
|---|---|---|
| `target/ast10x0/peripherals/smc/controller.rs` (`Smc<B, Mode>`, `Uninitialized`, `Ready`) | Type-state pattern with PhantomData | high |
| `target/ast10x0/peripherals/spimonitor/controller.rs` (`Uninitialized → Configured → Locked`) | Multi-stage lifecycle reusing the same shape | high |
| `target/ast10x0/peripherals/smc/fmc_backend.rs` (`PhantomData<UnsafeCell<()>>` for `!Sync`) | Single-threaded enforcement at the type level | high |
| `target/ast10x0/tests/smc/target.rs` (init flow `unsafe { UninitSmc::new(...) }.init()?`) | Pattern is exercised end-to-end | high |
| Repo-wide search for `Rc`/`Arc` in `target/ast10x0/` | Absence of shared-ownership primitives | high |

## 5. Scorecard Result

Not applicable — retrospective.

## 6. Decision

- Chosen option: Option A. Each peripheral is `Peripheral<Mode>` (or `Peripheral<Backend, Mode>`); init is a state-consuming method; `&self` for reads, `&mut self` for state-changing operations.
- Rationale: invariants the embedded context already guarantees (single-threaded, single instance per hardware block) become compile-time facts at zero runtime cost. Lifecycle stages with security weight (spimonitor's Locked) get the strongest possible enforcement: code that tries to mutate a Locked monitor does not compile.
- Why now: pattern is already pervasive across SMC, spimonitor, and the device facades. Documenting it retrospectively keeps new peripherals from drifting away from it.

## 7. Risks And Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
|---|---|---|---|---|
| New peripheral driver adopts a different ownership model and creates a second pattern | med | med | reference this DEC in CLAUDE.md / driver review checklist; require rationale to deviate | platform architecture |
| Future need for cross-task sharing forces a rewrite (e.g., RTOS task using the same peripheral) | low | high | revisit if multi-core or task-shared peripherals enter scope | platform architecture |
| `unsafe` constructor contract ("only one instance per hardware block") is convention, not type-enforced | med | med | board init code is the single allowed instantiation site; document in driver-level safety comments | firmware |
| Generic noise (`Smc<B, Mode>`) raises the bar for new contributors | med | low | type aliases (`UninitSmc`, `ReadySmc`) hide it at use sites | firmware |

## 8. Validation Plan

- Existing tests: `target/ast10x0/tests/smc/target.rs`, `target/ast10x0/tests/smc/target_device.rs`, `target/ast10x0/tests/spim/test_boot_uc/target.rs` all exercise the init → use → drop flow.
- Compile-time check: attempting `uninit.read(...)` should fail to compile (the read method is only on `Smc<B, Ready>`).
- Required for new drivers: at minimum a smoke test that walks the full state transition.

## 9. Revisit Trigger

- Revisit date: 2026-11-09
- Early revisit signals: multi-core or RTOS task-sharing requirements; introduction of any peripheral that genuinely needs runtime-checked sharing; need to support hot-reconfiguration that doesn't fit a typestate transition.

## 10. Copilot Audit Prompt

"Review this decision for unsupported claims, stale evidence, and missing failure modes. Provide concrete gaps with file links and suggested fixes."
