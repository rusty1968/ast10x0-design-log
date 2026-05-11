# Evidence Card: Controller-relative flash offset implicitly encodes chip select for DMA

- Evidence ID: EVD-20260511-dma-flash-offset-encodes-cs
- Date captured: 2026-05-11
- Captured by: copilot
- Source type: code + hardware register analysis
- Source links:
  - aspeed-rust/src/spi/fmccontroller.rs#read_dma
  - openprot/target/ast10x0/peripherals/smc/controller.rs#setup_segments
  - openprot/target/ast10x0/peripherals/smc/types.rs#flash_window_address
  - openprot/target/ast10x0/peripherals/smc/planning/dma-gap-fix-plan.md
- Related decision IDs:
  - DEC-20260511-smc-multi-cs

## Claim

A `dma_read` API that accepts a controller-relative `flash_offset` does not need a
`ChipSelect` argument. The offset unambiguously identifies both the target device and
the byte offset within it, making a separate `cs` parameter redundant and
potentially harmful.

## Observed Data

### Background: AHB window and memory-mapped flash

AHB (Advanced High-performance Bus) is the ARM AMBA bus that connects the CPU to
memory-mapped peripherals on the AST10x0 SoC. The SMC controller reserves a fixed
region of the CPU's physical address space — the **AHB flash window** — where flash
contents are directly visible as ordinary memory. The hardware SPI engine
automatically converts CPU load instructions into SPI read transactions on the wire
(execute-in-place / XIP mode). No driver code is involved; the CPU simply reads
from a physical address and flash bytes come back.

The three SMC controllers each occupy a separate 256 MB AHB window:

| Controller | AHB window base |
|---|---|
| FMC  | `0x8000_0000` |
| SPI1 | `0x9000_0000` |
| SPI2 | `0xB000_0000` |

The DMA engine uses a **different address base** for its flash address register
(`fmc084`): `SPI_DMA_FLASH_MAP_BASE = 0x6000_0000`. This is the DMA engine's own
view of flash address space, which is offset from the AHB window by `0x2000_0000`
for FMC. The formula `flash_window_base - SPI_DMA_FLASH_MAP_BASE + cs_offset`
converts from the AHB window address space to the DMA engine's flash address space;
both refer to the same physical flash bytes at the same byte offset.

### Hardware layout

The SMC controller maps each CS to a contiguous, non-overlapping slice of a single
256 MB AHB window. `setup_segments` programs segment registers at `init()` to
partition this window:

```
AHB window: [flash_window_base, flash_window_base + 256 MB)
  CS0:       [flash_window_base,             flash_window_base + cs0_size)
  CS1:       [flash_window_base + cs0_size,  flash_window_base + cs0_size + cs1_size)
```

For FMC: `flash_window_base = 0x8000_0000`.  
For SPI1: `flash_window_base = 0x9000_0000`.  
For SPI2: `flash_window_base = 0xB000_0000`.

### openprot offset convention

`dma_read(flash_offset, ...)` receives a controller-relative byte offset, identical
to what `read()` uses for AHB-mapped access. Valid range: `[0, capacity_bytes)` where
`capacity_bytes = cs0_size + cs1_size`. This is enforced by `validate_mapped_range`.

### CS derivation from offset

Because the CS slices are adjacent and non-overlapping:

```
flash_offset < cs0_size   →   target is CS0,  cs_offset = flash_offset
flash_offset ≥ cs0_size   →   target is CS1,  cs_offset = flash_offset − cs0_size
```

This derivation is lossless: given `flash_offset` and `cs0_size`, the target CS and
the within-CS byte offset are both uniquely recoverable. No information is discarded.

### DMA flash address register formula

`aspeed-rust fmccontroller.rs::read_dma` programs `fmc084` as:

```rust
let flash_start = decode_addr[cs].start + op.address.value - SPI_DMA_FLASH_MAP_BASE;
```

where `decode_addr[cs].start` is the AHB base address for the selected CS
(`flash_window_base[cs]`), `op.address.value` is the within-CS byte offset, and
`SPI_DMA_FLASH_MAP_BASE = 0x6000_0000` is the DMA engine's flash address base.

In openprot terms, this becomes:

```
fmc084 = flash_window_base[cs] - SPI_DMA_FLASH_MAP_BASE + cs_offset
```

The CS index and `cs_offset` are derived from `flash_offset` and `cs0_size` as shown
above. The DMA engine receives the correct flash address without the caller ever
naming a CS.

## Interpretation

### What this evidence strongly supports

The concept of "chip select" does appear inside `validate_dma_read` — as a local
variable `cs` (value 0 or 1) derived from `flash_offset` and `cs0_capacity`. It is
used to index `flash_window_base[cs]` and produce the correct `fmc084` value. This
is intentional and necessary.

What the evidence shows is that `cs` must **not cross the API boundary** as a
caller-supplied argument to `dma_read`. Any caller that constructs a `flash_offset`
in `[0, capacity_bytes)` has already implicitly selected a CS — the offset arithmetic
alone determines which device the hardware will address. The derivation is performed
once, internally, and the result is used immediately to compute the register value.
Accepting both `flash_offset` and a `cs` argument from the caller would be
**strictly redundant**: either the `cs` argument is consistent with the offset (in
which case it adds nothing), or it is inconsistent (in which case the correct
behavior is undefined — use the offset? use `cs`?). Neither case benefits from the
extra argument; the inconsistent case creates a latent bug class the type system
cannot catch.

### What a `cs` argument would add (and why it is harmful)

If `dma_read` accepted both `flash_offset` and `cs`, two sources of truth would
exist for which device is targeted. A caller could pass `flash_offset = 0x200`
(within CS0's region) with `cs = Cs1`. The correct behavior in that case is
undefined: either the register formula ignores `cs` (making the argument pointless),
or it overrides the offset arithmetic (selecting the wrong flash address). Neither
is desirable. The redundancy creates an ambiguity class that cannot be caught by
the type system, only by a runtime consistency check — which would itself be
surprising to callers who believe the offset already determines the target.

### What this evidence does not prove

- That aspeed-rust's blocking DMA model should be adopted (openprot uses a
  non-blocking model per other ADRs).
- That the CS derivation is correct for write or erase paths (DMA write and
  erase are out of scope for this analysis).

## Confidence

- Level: high
- Reason: the flash address formula is directly observed in aspeed-rust source;
  the offset-to-CS derivation follows from the non-overlapping segment layout
  enforced by `setup_segments`; no hardware ambiguity is possible given a valid
  `flash_offset`.
- Recency: analysis performed 2026-05-11 against current openprot `controller.rs`
  and aspeed-rust `fmccontroller.rs`.

## Counter-Evidence

- Known conflicting evidence IDs: none
- Notes on conflict resolution: n/a

## Reuse Tags

- Components: smc, dma, fmc, spi1, spi2
- Risk area: register correctness, API design
- Keywords: flash_offset, chip_select, dma_read, fmc084, flash_window_base, SPI_DMA_FLASH_MAP_BASE
