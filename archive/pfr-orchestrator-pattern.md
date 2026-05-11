# The PFR Orchestrator: Implementation Pattern

## 1. Purpose and Scope

NIST SP 800-193 defines *what* platform firmware resiliency must do — protect, detect, recover — but it deliberately does not prescribe *how*. It describes Roots of Trust (RTU, RTD, RTRec), Chains of Trust, and host/symbiont relationships, and leaves the architecture to implementers.

In practice, a dominant industry pattern has emerged in server and high-assurance platforms: a single **hardware orchestrator** (typically a CPLD or FPGA) that physically sits between firmware sources and target flash devices, owns all three Roots of Trust, and mediates every update, boot, and recovery action.

This document describes that pattern: its responsibilities, topology, state machine, and common implementation variants. It is intended for platform architects, firmware engineers, and security reviewers evaluating or designing a PFR-compliant platform.

The term **PFR orchestrator** is used here to mean the central hardware enforcement point. It is *not* a NIST term — NIST defines RoTs and CoTs, not orchestrators. Vendor literature variously calls this component a *Hardware Root of Trust (HRoT)*, *Platform Root of Trust*, *PFR CPLD*, or *security FPGA*.

---

## 2. Mapping to NIST SP 800-193

The orchestrator collapses what the standard treats as logically distinct entities into a single physical component:

| NIST 800-193 entity | Orchestrator role |
|---|---|
| Root of Trust for Update (RTU) | Verifies digital signatures on incoming firmware capsules before any flash write |
| Root of Trust for Detection (RTD) | Computes/verifies hashes of staged firmware regions on every boot |
| Root of Trust for Recovery (RTRec) | Owns the golden recovery image(s) and drives the rewrite process |
| Chain of Trust (CoT) | Enforces ordered, gated boot release of downstream devices (BMC, then BIOS, then CPU) |
| Host device (for symbiont firmware) | Acts as host for symbiont devices that lack their own RoT (PSU, NIC option ROM, etc.) |
| Critical Platform Device | Is itself a critical platform device — often *the* most critical |

The architectural advantage is unification: one component, one attack surface, one set of keys, one audit boundary. The cost is that the orchestrator becomes a single point of trust whose own integrity must be guaranteed by other means (immutable boot ROM, fused keys, anti-rollback counters).

---

## 3. Reference Topology

```
                       +--------------------------+
                       |   External update path   |
                       | (BMC OOB, OS in-band, USB)|
                       +------------+-------------+
                                    |
                                    v
                          +---------+---------+
                          |   PFR ORCHESTRATOR |
                          |   (CPLD or FPGA)   |
                          |                    |
                          |  - RTU / RTD / RTRec|
                          |  - SPI filter       |
                          |  - SMBus filter     |
                          |  - Key store (UFM)  |
                          |  - Anti-rollback SVN|
                          |  - Mailbox / telemetry
                          +--+----+----+----+--+
                             |    |    |    |
                  SPI mux/   |    |    |    |   power-good /
                  filter to  |    |    |    |   reset gating
                             v    v    v    v
                       +-----+ +--+-+ +-+--+ +-+-----+
                       | BMC | |BIOS| |PCH | |  CPU  |
                       |Flash| |Flas| |Desc| | (held |
                       +-----+ +----+ +----+ |in rst)|
                                              +------+
                       +-------+   +-------+  +-------+
                       | Golden|   | Golden|  | Active|
                       |  BMC  |   |  BIOS |  | regions
                       | image |   | image |  +-------+
                       +-------+   +-------+
```

Key topological properties:

- **Inline interposer.** The orchestrator physically owns the SPI bus to BMC and BIOS flash. The BMC and CPU/PCH cannot reach their own boot flash except through it.
- **Reset and power gating.** The orchestrator holds downstream devices in reset until verification completes. This is what makes the Chain of Trust enforceable rather than advisory.
- **Independent storage.** Golden recovery images live in flash regions only the orchestrator can write — typically a separate region of the same flash, or a dedicated recovery flash, never reachable by the BMC or host CPU.
- **Sideband filtering.** SMBus/I²C traffic to sensitive components (PMICs, VR controllers, NVRAM holding the PIT password) is filtered against an allowlist.

---

## 4. Core Responsibilities

### 4.1 Authenticated Update (RTU)

Every firmware capsule arriving at the orchestrator — whether for the BMC, BIOS, the orchestrator itself, or a symbiont device — passes through this pipeline:

1. **Capsule format check.** Header magic, length, region targets, manifest layout.
2. **Signature verification.** Typically ECDSA P-256 or P-384 against a public key (or hash of a public key) burned into the orchestrator's user flash (UFM) or eFuses. Some designs implement a key hierarchy: a root key signs intermediate keys, which sign capsules.
3. **Anti-rollback check.** Each capsule carries a Security Version Number (SVN). The orchestrator maintains a monotonic SVN per region and rejects capsules with an SVN lower than the stored value. This blocks reinstallation of a known-vulnerable signed image.
4. **Staging.** Verified capsules are written to a staging region, *not* directly to the active region.
5. **Promotion.** On the next reset, the orchestrator copies staging → active, then re-verifies before releasing the device from reset.

A capsule that fails any step is dropped, logged, and never reaches the target device.

### 4.2 Boot-time Detection (RTD)

On every power-on (the **T-1 phase**, see §5):

1. CPU, BMC, and PCH are held in reset.
2. The orchestrator reads each protected region of BMC and BIOS flash.
3. It computes a hash and compares to the manifest signature stored alongside the image.
4. Pass → release the device from reset. Fail → trigger recovery.

This is the practical realization of NIST's RTD requirement that detection occur before code is executed by the device under inspection.

### 4.3 Recovery (RTRec)

When detection fails, or when the device fails to reach a heartbeat checkpoint within a watchdog window:

1. The orchestrator erases the corrupted active region.
2. It copies the golden image from its private recovery store.
3. It re-verifies the signature on the recovered image (defense in depth — recovery store could itself be corrupted by hardware fault).
4. It releases the device from reset and observes the boot.

Recovery is automatic by default but can be policy-gated (e.g., require physical presence, or require manual administrator initiation in regulated environments — see NIST 800-193 §3.6.4).

### 4.4 Runtime Protection

Protection does not stop at boot. The orchestrator continuously:

- **Filters SPI transactions** against an allowlist of opcodes and address ranges. Writes to read-only regions (BIOS region containing host boot code, key manifests, the orchestrator's own update region) are silently dropped or NAK'd.
- **Filters SMBus/I²C transactions** to sensitive components.
- **Watches heartbeat signals** from the BMC and host firmware. Loss of heartbeat → recovery.
- **Logs events** to a mailbox interface readable by the BMC (and via the BMC, by remote management).

---

## 5. Platform State Machine

Industry implementations converge on a three-state model that maps cleanly onto NIST's protect/detect/recover principles:

| State | Name | What's happening | Who is running |
|---|---|---|---|
| **T-1** | Pre-boot verification | Orchestrator is awake; everything else is held in reset. RTD verifies BMC and BIOS images. RTRec runs if verification fails. | Orchestrator only |
| **T0** | Operational / runtime | All devices released, OS running. Orchestrator filters SPI/SMBus, watches heartbeats, accepts updates into staging. | Full platform |
| **T-min** (sometimes) | Critical degraded mode | Used when only the orchestrator and minimal subsystems are trusted; e.g., post-recovery before full reboot. | Orchestrator + minimal devices |

Transitions:

- **AC power-on** → T-1
- **T-1 verification pass** → T0
- **T-1 verification fail** → recovery → T-1 (re-verify) → T0
- **T0 heartbeat loss / runtime detection event** → forced reset → T-1
- **Capsule promotion** → forced reset → T-1

The discipline is: nothing transitions to T0 until the orchestrator has personally verified every protected region.

---

## 6. Operational Flows

### 6.1 Authenticated update flow (in-band, OS-driven)

```
OS update agent ──capsule──> BMC ──SPI passthrough──> Orchestrator
                                                          │
                                                          ▼
                                              [verify signature]
                                              [check SVN]
                                              [write to staging]
                                                          │
                                                          ▼
                                              [acknowledge to BMC]
                                                          │
                                                          ▼
                                              [next reset triggers
                                               promotion + re-verify]
```

### 6.2 Recovery flow (automatic)

```
T-1: orchestrator hashes BIOS active region
        │
        ▼
   hash mismatch
        │
        ▼
   [erase BIOS active region]
        │
        ▼
   [copy from golden region]
        │
        ▼
   [re-hash, verify signature]
        │
        ▼
   [release CPU from reset]
        │
        ▼
   [log event to mailbox; BMC reads, surfaces to management plane]
```

### 6.3 Orchestrator self-update

The orchestrator must be able to update its own firmware without bricking the platform. The typical pattern:

1. Capsule arrives, signed with a key dedicated to orchestrator updates (separate from BMC/BIOS keys).
2. Verified capsule written to the orchestrator's *inactive* configuration image (CFM1 vs CFM0 in Lattice/Intel parlance).
3. On reset, the orchestrator boots from the new image. If boot fails (immutable boot ROM detects bad image), it falls back to the previous image automatically.

The immutable boot loader inside the orchestrator silicon is the ultimate root — it cannot be field-updated.

---

## 7. Trust Boundaries and Threat Model

What the orchestrator *defends against*:

- Compromised BMC firmware attempting to write the BIOS region or its own boot region.
- Compromised host OS attempting to flash BIOS or BMC out-of-policy.
- Replay of an older signed image with known vulnerabilities (anti-rollback).
- Corruption of flash due to power loss, bit flips, failed updates.
- Limited classes of physical attack on the SPI bus (filtering blocks rogue writes from anything not the orchestrator).

What it *does not* defend against without additional measures:

- Physical replacement of the orchestrator silicon itself.
- Compromise of the signing keys at the vendor.
- Side-channel attacks on the orchestrator's cryptographic operations.
- Attackers with sustained physical access and lab equipment (decapping, probing).
- Bugs in the orchestrator's own firmware/RTL.

The orchestrator's trustworthiness rests on: (a) immutable boot ROM, (b) fused or UFM-stored keys that cannot be rewritten in the field, (c) a small, auditable codebase, and (d) physical tamper resistance appropriate to the threat model.

---

## 8. Implementation Variants

| Variant | Typical use | Pros | Cons |
|---|---|---|---|
| **Discrete CPLD** (e.g., Lattice MachXO3D, Mach-NX) | Mainstream servers, OCP designs | Low cost, low power, deterministic, small attack surface | Limited compute — slow ECDSA, limited algorithm agility |
| **Low-end FPGA with soft CPU** (e.g., Intel/Altera MAX 10 + Nios II — used by Intel PFR) | Intel reference platforms, Xeon Scalable boards | CPLD-class form factor and instant-on, but FPGA flexibility; dual-image internal flash gives fail-safe self-update | Vendor-locked toolchain (Quartus); Nios II ecosystem narrower than ARM |
| **Discrete security FPGA** (e.g., Lattice Avant, Microchip PolarFire) | High-end / high-assurance servers | More crypto headroom, post-quantum upgrade path, more I/O | Higher cost and power |
| **Purpose-built PFR SoC** (e.g., ASPEED AST1060, Microchip CEC1736) | DC-SCM 2.0 modules, ASPEED-BMC ecosystems, edge/embedded | Rich crypto libraries, mature toolchains (Zephyr, mbedTLS), hard-IP CPU + hardware bus-filter blocks, on-die flash; faster crypto than CPLDs; algorithm-agile | Larger TCB than RTL-only designs — Zephyr/mbedTLS CVEs become RoT CVEs |
| **CPU microcode-only** (vendor-internal RoT) | Some client platforms | No extra BoM | Limited scope — protects boot firmware only, not BMC or symbionts |

Selection is usually driven by: cost target, cryptographic agility requirements (post-quantum readiness pushes toward FPGA), the number of flash devices and sideband buses to mediate, and the customer's certification needs (FIPS 140-3, Common Criteria, OCP S.A.F.E.).

### 8.1 Silicon vs. capsule format — they are independent choices

A subtle but important point: the choice of orchestrator **silicon** is independent from the choice of **capsule format** the orchestrator consumes. The capsule format (manifest layout, signature scheme, region descriptors, SVN encoding) is the interop contract between the firmware build/sign infrastructure and the orchestrator. The silicon and firmware underneath are an implementation detail.

In practice, two formats dominate:

- **Intel PFR capsule format** — defined by Intel's open-source reference, used on essentially all Intel-platform PFR designs.
- **AMD PFR capsule format** — AMD's equivalent for EPYC platforms.

ASPEED's Zephyr stack ships board configurations for both, selectable at build time. This means:

- An Intel-platform DC-SCM 2.0 module can use either a **MAX 10 FPGA** (Intel reference firmware) **or** an **AST1060 SoC** (ASPEED Zephyr or AMI Tektagon configured for Intel image format) as the orchestrator. Both are NIST 800-193 compliant. Both consume Intel-PFR-format capsules. The rest of the platform can't tell the difference at the capsule layer.
- The same AST1060 silicon can be reflashed with the AMD-format build to serve an EPYC platform.

This decoupling is what makes "is X PFR-compliant?" a less useful question than "what silicon, what firmware, and which capsule format?" — the three need to be answered separately.

---

## 9. Reference Implementations

- **Intel PFR** — original commercial implementation, Xeon Scalable platforms (3rd-gen and later). FPGA-based on Intel/Altera **MAX 10** with a **Nios II soft core**; defines the T-1/T0 state model and the PIT (Platform Integrity Technology) password levels. Reference for SPI/SMBus filtering. Open-sourced at `github.com/intel/platform-firmware-resiliency`. Defines the canonical PFR capsule format used as the de-facto interop contract by other implementations.
- **ASPEED PFR (AST1060)** — purpose-built **PRoT SoC** (ARM Cortex-M4F at 200 MHz) with on-die flash/SRAM, hardware crypto engine (AES, SHA, RSA, ECDSA — FIPS-validated), and dedicated **QSPI Monitor** and **SMBus Filter** hardware blocks. Reference firmware is Zephyr-based (`github.com/AspeedTech-BMC/aspeed-zephyr-project`) and ships with board configurations for **Intel PFR image format**, **AMD PFR image format**, and **DICE**-enabled variants. Commonly paired with the AST2600/AST2700 BMC on DC-SCM 2.0 modules. Note: the **AST2600 BMC's own** built-in TrustZone/OTP RoT protects the BMC's boot chain only — it is *not* a platform-wide PFR orchestrator and does not replace the AST1060 in this role.
- **Lattice Sentry / Mach-NX solution stack** — turnkey FPGA-based PFR for OEMs; widely deployed.
- **AMI Tektagon (formerly PlatFire) / Tektagon XFR** — firmware stack productizing the orchestrator pattern; runs on Lattice FPGAs *and* on the ASPEED AST1060, giving OEMs silicon flexibility under one firmware vendor.
- **Insyde Supervyse** — BMC management firmware with AST1060 HRoT support; another commercial AST1060 firmware option.
- **Microchip CEC1736 Trust Shield** — MCU-based variant, positioned more broadly than PFR (also used for general platform attestation).
- **OCP S.A.F.E.** (Security Appraisal Framework and Enablement) — Open Compute Project's security review framework; PFR orchestrator designs are commonly evaluated against it.
- **OCP Cerberus** — *related but distinct*: Cerberus is primarily an attestation root of trust (challenge/response, measurement reporting), whereas the PFR orchestrator is primarily an enforcement root of trust. Some designs combine both roles in one component.

---

## 10. Design Considerations and Common Pitfalls

**Key management is the hard part.** Signing infrastructure, key rotation, key revocation, and the relationship between manufacturer keys and owner keys are usually under-specified in initial designs. Decide early: who can sign? How are compromised keys revoked? Is there an owner-controlled signing slot (often required by hyperscaler customers)?

**Anti-rollback hurts as much as it helps.** SVN bumps must be coordinated across the BMC, BIOS, and orchestrator. A field unit that boots an old SVN cannot be downgraded — including for legitimate reasons like reproducing a customer issue. Plan for this operationally.

**Recovery image staleness.** The golden image is set at manufacturing and is typically not updated in the field (because updating it defeats its purpose as an immutable known-good). This means the recovery image may carry vulnerabilities patched in active firmware. Mitigations: ship recovery images with minimum SVN, require update-after-recovery, or implement a "blessed update" mechanism that can advance the recovery slot under stricter controls.

**Heartbeat and watchdog tuning.** Too tight → false-positive recoveries during legitimate long-running operations (memory training, BIOS POST on large-memory systems). Too loose → attacker has a long window to operate. This is typically platform-specific and discovered empirically.

**Telemetry and observability.** The orchestrator's mailbox is the only window into its decisions. Insufficient logging = unattributable production incidents. Log: every verification result, every filter drop with opcode/address, every recovery event, every SVN bump.

**Don't let the BMC become the orchestrator's master.** It's tempting to drive orchestrator state through the BMC for convenience. Resist. The orchestrator must be able to recover the BMC, which means it cannot depend on the BMC for its own correct operation. The BMC reads from the orchestrator; it does not command it (with narrow, authenticated exceptions).

**Plan the immutable-vs-mutable boundary deliberately.** Everything mutable can be attacked; everything immutable can never be patched. Typical split: the orchestrator's first-stage boot ROM and the root signing key hash are immutable; the orchestrator's main firmware, the recovery images, and the active firmware regions are mutable but signature-verified.

---

## 11. Glossary

- **CFM (Configuration Flash Memory)** — Lattice/Intel term for the orchestrator's own bitstream storage, typically dual-image (CFM0/CFM1) for fail-safe self-update.
- **CoT** — Chain of Trust (NIST 800-193).
- **DC-SCM** — Data Center Secure Control Module; OCP-defined modular form factor that places the BMC + PRoT on a swappable card. PFR orchestrator commonly lives here.
- **DICE** — Device Identifier Composition Engine (TCG); a layered measurement/identity scheme that some PFR orchestrators (e.g., AST1060 with MCUboot) implement on top of the base PFR functions.
- **Golden image** — The known-good firmware image used by RTRec for recovery.
- **HRoT** — Hardware Root of Trust; vendor synonym for the orchestrator.
- **MCUboot** — Open-source secure bootloader commonly used as the immutable first stage on MCU-based orchestrators (e.g., AST1060 Zephyr stack).
- **PIT** — Platform Integrity Technology; Intel's anti-tamper extensions on top of base PFR.
- **PRoT** — Platform Root of Trust; vendor term (especially ASPEED, AMI) for the orchestrator silicon and/or firmware.
- **RoT** — Root of Trust (NIST 800-193): RTU, RTD, RTRec.
- **SVN** — Security Version Number; monotonic counter used for anti-rollback.
- **T-1 / T0 / T-min** — Platform states (pre-verify, runtime, degraded).
- **UFM (User Flash Memory)** — On-die non-volatile storage in CPLDs/FPGAs; typically holds keys, SVN counters, manifests.

---

## 12. References

- NIST SP 800-193, *Platform Firmware Resiliency Guidelines* (May 2018) — https://doi.org/10.6028/NIST.SP.800-193
- Intel, *Platform Firmware Resilience* technical overview
- Intel, *platform-firmware-resiliency* open-source reference — https://github.com/intel/platform-firmware-resiliency
- ASPEED, *AST1060 Platform Root of Trust Processor* product page
- ASPEED, *aspeed-zephyr-project* — https://github.com/AspeedTech-BMC/aspeed-zephyr-project
- Lattice Semiconductor, *Platform Firmware Resiliency* solution brief
- Open Compute Project, *S.A.F.E.* and *Cerberus* specifications
- Open Compute Project, *DC-SCM 2.0* specification
- TianoCore documentation, *Understanding UEFI Secure Boot Chain* — chapter on PFR implementations
