---
layout: post
title:  Arm System Control Processor Firmware (SCP-firmware)
categories: [ARM, Firmware, Boot, Embedded]
---

## INTRO

Almost every modern Arm SoC contains a processor you rarely think about. It is not one of the application cores. It boots before they do, it decides whether they get power at all, and it keeps running long after Linux is up — arbitrating clocks, voltages, thermal limits and CPU hotplug requests.

That processor is the <span class="emphasizer_code_function_2">System Control Processor</span> (__SCP__), and the Power Control System Architecture ([__DEN0050C__](https://developer.arm.com/documentation/den0050/d/?lang=en)) describes the role it plays. Arm ships a reference implementation of its firmware: [__SCP-firmware__](https://github.com/ARM-software/SCP-firmware).

This post is a source-level teardown of the public upstream snapshot [`b34e5ce`](https://github.com/ARM-software/SCP-firmware/tree/b34e5ce) (SCP-firmware __v2.14.0__) — how the codebase is structured, how the firmware boots, and how a small Cortex-M microcontroller physically brings a cluster of Cortex-A cores out of reset.

Let's go:

- [What Runs Where](#what-runs-where)
<br>
- [Layer Stack](#layer-stack)
<br>
- [Framework Core](#framework-core)
<br>
- [Build System and Code Generation](#build-system-and-code-generation)
<br>
- [Boot Sequence](#boot-sequence)
<br>
- [The Mutual Bootstrap](#the-mutual-bootstrap)
<br>
- [Hardware Access](#hardware-access)
<br>
- [Reset Vectors](#reset-vectors)
<br>
- [SCMI at Runtime](#scmi-at-runtime)
<br>
- [Observations](#observations)
<br>
- [References](#references)



## What Runs Where

The __SCP__ is a separate microcontroller sitting on the SoC alongside the application processors. This firmware runs on that microcontroller. The Cortex-A cores are what it _manages_ — it never executes on them.

The Cortex-M products examined here set <span class="emphasizer_code_function_2">SCP_ARCHITECTURE "arm-m"</span>:

| Product | Core | Profile |
|---|---|---|
| Juno | Cortex-M3 | ARMv7-M |
| Total Compute tc2 | Cortex-M3 | ARMv7-M |
| Morello (SCP + MCP) | Cortex-M7 | ARMv7-M |

<br>
The Cortex-M target explains nearly every design decision that follows: a cooperative single-threaded event loop rather than an __RTOS__, a fixed preallocated event pool, allocate-only memory, some framework event validation compiled out of release builds, and `WFI` in the idle path.

The upstream tree also supports other execution environments, including `arm/aarch64`, `none/host`, `none/posix`, and `optee`. This post focuses on the Cortex-M products above.

_Note also_ that a single SoC can carry more than one of these. Morello ships both an __SCP__ and an <span class="emphasizer_code_function_2">MCP</span> (Manageability Control Processor), each a Cortex-M7, each with its own ROM and RAM firmware images built from this same tree.


## Layer Stack

Roughly __378k__ lines of C across 686 `.c` and 697 `.h` files, in five layers.

| Layer | Path | Role |
|---|---|---|
| Framework | `framework/` | ~10.8k LOC. Event loop, module lifecycle, identifiers, lists, logging, IO, allocation |
| Arch | `arch/` | `arm/arm-m`, `arm/aarch64`, `none/posix`, `none/host`, `optee` |
| Modules | `module/` | 110 generic, reusable units |
| Products | `product/` | Platform-specific firmware targets, including juno, morello and totalcompute |
| Interfaces | `interface/` | Header-only cross-module contracts |

<br>
<span class="emphasizer_code_function_2">SCMI</span> is the dominant feature area by volume — a core `scmi` dispatcher (9.5k LOC) plus roughly fifteen protocol modules. `scmi_perf` alone is __18k__ LOC, the largest module in the tree. Several protocols also ship `_req` requester variants, meaning the SCP can act as an SCMI _agent_ toward another controller, not only as the platform side.


## Framework Core

Everything is a module. A module is a `struct fwk_module` descriptor plus optional _elements_ and _sub-elements_. Four declared types — <span class="emphasizer_code_function_2">HAL</span>, <span class="emphasizer_code_function_2">DRIVER</span>, <span class="emphasizer_code_function_2">PROTOCOL</span>, <span class="emphasizer_code_function_2">SERVICE</span> — which are purely informational; the framework treats them identically.

### Entity Identifiers

Every entity in the system is addressed by a packed 32-bit identifier. One word — cheap to pass, compare and log.

```
type:4 │ module_idx:8 │ element_idx:12 │ sub_element_idx:8
                      │ api_idx:4      │ reserved:16
                      │ event_idx:6    │ reserved:14
                      │ notification_idx:6
```

The bitfield widths in `framework/include/internal/fwk_id.h` are hard architectural ceilings:

| Entity | Bits | Ceiling |
|---|---|---|
| Modules | 8 | 256 per system |
| Elements | 12 | 4096 per module |
| Sub-elements | 8 | 256 per element |
| APIs | 4 | 16 per module |
| Events | 6 | 64 per module |
| Notifications | 6 | 64 per module |

<br>

### Lifecycle

Startup runs four ordered phases. _Binding_ is where modules exchange API pointers — `fwk_module_bind(target_id, api_id, &api_ptr)`, validated against the target's declared `api_count`.

```
init modules → init elements → bind round 0 → bind round 1 → start → event loop
```

<span class="emphasizer_code_function_2">FWK_MODULE_BIND_ROUND_MAX</span> is __1__. Two rounds, no more. Round 0 handles module-to-module binds, round 1 handles element binds and late dependencies. A dependency chain needing three levels of late binding has to be restructured or deferred to the start phase.

### The Event Loop

Single-threaded, cooperative, __run-to-completion__. No RTOS, no preemption between handlers, and consequently _no locking anywhere_ in the module code.

- Two queues: `event_queue` for thread context, `isr_event_queue` for interrupt context.
- A free list of preallocated `fwk_event` structs, sized `FWK_MODULE_EVENT_COUNT`. It never grows.
- Only free-list operations and ISR-queue access wrap in `fwk_interrupt_global_disable()`. The main queue needs no guard because __ISRs never touch it__.
- The loop drains the main queue, migrates ISR events across, flushes the log, then calls `arch_suspend()` — `WFI`/`WFE`.
- `fwk_event_light` is a parameter-less event variant, added for footprint.

Pool exhaustion logs `FWK_LOG_CRIT`, calls `fwk_unexpected()` and returns `FWK_E_NOMEM`. One module leaking events starves the entire system.

### Deferred Responses

The asynchronous spine. A __HAL__ returns <span class="emphasizer_code_function_2">FWK_PENDING</span>; the response event is parked on a per-entity delayed-response list and matched later by `cookie`.

The caller must _already_ be inside its own event context to receive the reply — which is why modules frequently send an event to themselves before calling a HAL. So a module's `start()` often does nothing but post one event to itself, because the framework's start phase must complete for _every_ module before the event loop begins running.


## Build System and Code Generation

There is __no dynamic registration anywhere__. A firmware's `Firmware.cmake` lists `SCP_MODULES`, and `framework/CMakeLists.txt` generates the wiring from templates:

- `fwk_module_idx.h` — `enum fwk_module_idx`, `FWK_MODULE_ID_X` macros, per-module id constants
- `fwk_module_list.c` — `module_table[]` and `module_config_table[]`

Per-firmware `config_<module>.c` files supply each `const struct fwk_module_config`. Everything resolves at link time; unused modules are never compiled.

The __order__ of the module list _is_ the init, bind and start order. Total Compute tc2's `Firmware.cmake` states it plainly:

<br>
<hr class="line_1">
_The order of the modules in the following list is the order in which the modules are initialized, bound, started during the pre-runtime phase. any change in the order will cause firmware initialization errors._
<hr class="line_1">
<br>

It is a topological sort maintained _by hand_ for each firmware target.

Configurability is broad: 40-plus `BUILD_HAS_*` flags plus `SCP_ENABLE_*` CMake options covering notifications, resource permissions, fast channels, plugin handlers, in-band and out-of-band messaging, __ATU__ management, __IPO__ and newlib-nano. The combination matrix is far larger than what CI actually validates.


## Boot Sequence

<span class="emphasizer_code_function_2">scp_romfw</span> and <span class="emphasizer_code_function_2">scp_ramfw</span> are __two separate binaries built from the same tree__, differing only in module list, config files and linker memory layout. The framework startup path therefore runs _twice_ — once in ROM, once again in RAM after the jump.

```c
/* product/juno/include/scp_mmap.h */
#define SCP_ROM_SIZE  (64 * 1024)
#define SCP_ROM_BASE  0x00000000    /* FVP; board variant is 0xABE40000 */
#define SCP_RAM_SIZE  (128 * 1024)
#define SCP_RAM_BASE  0x10000000
```

### Stage 0 — Reset

The Cortex-M fetches its reset vector from ROM. CMSIS startup runs, then `main()` sets up the Configuration Control Register — trap divide-by-zero, enable stack alignment — calls the weak `platform_init_hook()`, and enters `fwk_arch_init()`. The framework takes over. A-cores are still in reset.

### Stage 1 — ROM Firmware Powers the A-cores

The ROM module list is deliberately small — no SCMI, no power-domain HAL, both far too heavy for 64 KiB. A _product-specific_ module drives the sequence:

- __Total Compute__: broadcast a `SYSTOP` notification with `response_requested`, count the outstanding subscribers, and when the count reaches zero zero the AP context area and ping the <span class="emphasizer_code_function_2">RSS</span> (Runtime Security Subsystem). On the RSS acknowledgement, power the primary cluster then the primary core, then load the image.
- __Juno__: same shape without the RSS. Walk a boot-map bitmap — cluster PPU on and wait, then each core whose bit is set — for _LITTLE_ and _big_ separately, switch each cluster's clock from the boot clock to its private __PLL__, reload the watchdog, load the image.

_Worth noting the naming:_ Total Compute tc2 calls this module `tc2_bl1`, borrowing __TF-A's__ BL1/BL2/BL31 convention. It is _not_ TF-A BL1 — it is an SCP-side module playing the analogous first-stage role on a different processor. Juno's equivalent is simply called `juno_rom`.

The split of responsibility is clean. The generic `bootloader` module in `module/bootloader/` does the portable work — SDS wait, limited image-metadata checks, byte copy, VTOR jump — while the product module holds the platform choreography: _which_ PPUs, in what order, clock switching, the RSS ping, the watchdog.

### Stage 2 — The SDS Handshake

The SCP __does not know where its own runtime image lives__. It spins on a valid bit in <span class="emphasizer_code_function_2">SDS</span> (Shared Data Storage), a shared-memory region of versioned, ID-tagged structures, waiting for AP-side firmware to publish the image location:

```c
/* module/bootloader/src/mod_bootloader.c */
while (true) {
    status = sds_api->struct_read(sds_struct_id,
                                  BOOTLOADER_STRUCT_VALID_POS,
                                  &image_flags, sizeof(image_flags));
    if (status != FWK_SUCCESS) {
        return status;
    }
    if ((image_flags & (uint32_t)IMAGE_FLAGS_VALID_MASK) != (uint32_t)0) {
        break;
    }
}
```

Once the bit is set it _clears the flag_ — so a warm reboot with retained RAM cannot replay a stale image — then reads the offset and size. It performs limited metadata checks: a nonzero size, a 4-byte-aligned offset, an offset-only source-region check, and destination capacity. It does not validate that the complete `offset + size` range lies in the source region.

### Stage 3 — The Jump

Disable interrupts globally (the vector table is about to be overwritten by the copy), flush the log, then hand off to a short assembly routine:

```armasm
/* module/bootloader/src/mod_bootloader_boot.S */
mod_bootloader_boot:
    movs r4, r0          /* save destination - soon the vector table */
1:  ldrb r5, [r1], #1    /* byte copy: AP memory -> SCP SRAM */
    strb r5, [r0], #1
    subs r2, #1
    bne  1b
    str  r4, [r3]        /* destination -> SCB->VTOR */
    ldr  r0, [r4]        /* vector[0] = initial MSP */
    msr  msp, r0
    ldr  r0, [r4, #4]    /* vector[1] = reset handler */
    bx   r0              /* "... and take a leap of faith" */
```

Standard M-profile relocation: retarget the vector table, reset the stack pointer, branch. The function is `noreturn` — everything after it in the caller is the failure path. 

### Stage 4 — RAM Firmware Runtime

The framework starts over with the full module set — power-domain HAL, system-power, SCMI and its protocols, DVFS, PSU, clocks.

The platform's system module subscribes to the SCMI and SDS _initialized_ notifications, then issues one __composite__ power-domain request covering `SYSTOP`, the primary cluster and the primary core in a single call, re-establishing the power-domain module's view of what ROM already powered:

```c
/* product/totalcompute/common/module/tc_system/src/mod_tc_system.c */
return mod_ctx.mod_pd_restricted_api->set_state(
    FWK_ID_ELEMENT(FWK_MODULE_IDX_POWER_DOMAIN, 0),
    MOD_PD_SET_STATE_NO_RESP,
    MOD_PD_COMPOSITE_STATE(MOD_PD_LEVEL_2, 0, MOD_PD_STATE_ON,
                           PD_INIT_STATE_PRIMARY_CLUSTER,
                           PD_INIT_STATE_PRIMARY_CORE));
```

When SCMI and SDS have _both_ reported ready, it writes a feature-availability bit into SDS. That bit is the "__SCMI is live__" signal the PSCI agent has been waiting on.

### Stage 5 — Secondary Cores

The SCP stops initiating. Every subsequent core power-on, CPU hotplug, suspend and resume arrives as an SCMI message from __TF-A BL31__ and travels the path traced [further below](#scmi-at-runtime), for the life of the system.


## The Mutual Bootstrap

The ordering in stages 1 and 2 looks backwards. The ROM powers the A-cores _before_ loading its own runtime image. Why?

Because neither processor can finish booting alone.

The SCP ROM is __64 KiB__. It has no flash driver, no eMMC driver, no filesystem and no cryptography. `scp_ramfw` lives in the system boot flash, and only the AP side has the drivers and the keys to fetch and verify it. The ROM's own configuration says so outright:

```c
/* product/juno/scp_romfw/config_bootloader.c */
static const struct mod_bootloader_config bootloader_module_config = {
    .source_base      = TRUSTED_RAM_BASE,   /* AP memory! */
    .source_size      = 256 * 1024,
    .destination_base = SCP_RAM_BASE,       /* SCP's own SRAM */
    .destination_size = SCP_RAM_SIZE,
    .sds_struct_id    = (uint32_t)JUNO_SDS_BOOTLOADER,
};
```

`TRUSTED_RAM_BASE` is __AP memory__. The SCP reads its own runtime firmware out of the application processor's trusted SRAM. Something has to put it there first — and that something is TF-A, running on the very cores the SCP just powered.

<div style="border:1px solid #ddd; background:#fcfcfc; padding:14px 10px; margin:20px 0; overflow-x:auto;">
<svg viewBox="0 0 760 420" style="width:100%; height:auto; min-width:640px; font-family:'Courier New',Courier,monospace;" role="img" aria-label="Mutual bootstrap between SCP and application processor">
<text x="118" y="20" style="font-size:12px; font-weight:bold; fill:#404040;" text-anchor="middle">SCP (Cortex-M)</text>
<text x="642" y="20" style="font-size:12px; font-weight:bold; fill:#404040;" text-anchor="middle">AP (Cortex-A)</text>
<line x1="118" y1="30" x2="118" y2="404" stroke="#c8c8c8" stroke-width="1" stroke-dasharray="3 4"/>
<line x1="642" y1="30" x2="642" y2="404" stroke="#c8c8c8" stroke-width="1" stroke-dasharray="3 4"/>
<rect x="18" y="44" width="200" height="40" fill="#eee" stroke="#b0b0b0"/>
<text x="32" y="62" style="font-size:11px; fill:#222;">ROM fw running</text>
<text x="32" y="77" style="font-size:9.5px; fill:#666;">A-cores held in reset</text>
<path d="M218 62 L 540 62" stroke="#404040" stroke-width="1.3" fill="none" marker-end="url(#a1)"/>
<text x="370" y="50" style="font-size:9.5px; fill:#666;" text-anchor="middle">PWPR = ON on cluster PPU, then core PPU</text>
<rect x="542" y="40" width="200" height="58" fill="#eee" stroke="#b0b0b0"/>
<text x="556" y="62" style="font-size:11px; fill:#222;">AP ROM -&gt; BL1 -&gt; BL2</text>
<text x="556" y="78" style="font-size:9.5px; fill:#666;">reads SCP_BL2 from flash,</text>
<text x="556" y="92" style="font-size:9.5px; fill:#666;">authenticates it</text>
<rect x="238" y="198" width="284" height="64" fill="#ffeee9" stroke="#fe7e61"/>
<text x="380" y="218" style="font-size:12px; font-weight:bold; fill:#d9532f;" text-anchor="middle">Shared trusted SRAM</text>
<text x="380" y="236" style="font-size:9.5px; fill:#666;" text-anchor="middle">SDS region: image offset + size</text>
<text x="380" y="251" style="font-size:9.5px; fill:#666;" text-anchor="middle">+ valid flag</text>
<path d="M542 77 L 530 77 L 530 230 L 522 230" stroke="#404040" stroke-width="1.3" stroke-linecap="square" stroke-linejoin="miter" fill="none" marker-end="url(#a1)"/>
<rect x="542" y="198" width="200" height="64" fill="#eee" stroke="#b0b0b0"/>
<text x="624" y="218" style="font-size:11px; fill:#222;" text-anchor="middle">BL2 writes image here</text>
<rect x="18" y="198" width="200" height="64" fill="#eee" stroke="#b0b0b0"/>
<text x="32" y="218" style="font-size:11px; fill:#222;">spinning on valid bit</text>
<text x="32" y="236" style="font-size:9.5px; fill:#666;">while (!(flags &amp; VALID))</text>
<text x="32" y="251" style="font-size:9.5px; fill:#666;">no timeout</text>
<path d="M238 230 L 218 230" stroke="#404040" stroke-width="1.3" stroke-linecap="square" fill="none" marker-end="url(#a1)"/>
<rect x="18" y="290" width="200" height="58" fill="#eee" stroke="#b0b0b0"/>
<text x="32" y="308" style="font-size:11px; fill:#222;">copy -&gt; SCP SRAM</text>
<text x="32" y="325" style="font-size:9.5px; fill:#666;">set VTOR, reset MSP,</text>
<text x="32" y="339" style="font-size:9.5px; fill:#666;">branch to reset handler</text>
<path d="M300 263 L 300 304 L 218 304" stroke="#404040" stroke-width="1.3" stroke-linecap="square" stroke-linejoin="miter" fill="none" marker-end="url(#a1)"/>
<rect x="18" y="362" width="200" height="42" fill="#eee" stroke="#b0b0b0"/>
<text x="32" y="380" style="font-size:11px; fill:#222;">RAM fw: SCMI stack up</text>
<text x="32" y="395" style="font-size:9.5px; fill:#666;">writes "messaging ready"</text>
<path d="M218 382 L 542 382" stroke="#404040" stroke-width="1.3" fill="none" marker-end="url(#a1)"/>
<text x="407" y="375" style="font-size:9.5px; fill:#666;" text-anchor="middle">via SDS feature-availability bit</text>
<rect x="542" y="362" width="200" height="42" fill="#eee" stroke="#b0b0b0"/>
<text x="556" y="380" style="font-size:11px; fill:#222;">BL31 waiting</text>
<text x="556" y="395" style="font-size:9.5px; fill:#666;">for SCMI to come alive</text>
<defs><marker id="a1" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0 0 L7 3.5 L0 7 z" fill="#404040"/></marker></defs>
</svg>
</div>

Each side unblocks the other in turn. The A-cores are powered early but cannot get far without the SCP; the SCP gets its firmware from the A-cores.

There is a trust consequence worth stating plainly: the SCP executes an image handed to it by the AP after only the limited metadata checks above. It does not authenticate or checksum the image. Authentication happened on the AP side, in __BL2__. The SCP trusts whoever can write SDS.


## Hardware Access

The SCP is a __bus master__ on the SoC interconnect. It does not _message_ the AP subsystem — it writes its registers and its memory directly.

### Powering a Core Is One Register Write

A-cores sit behind <span class="emphasizer_code_function_2">PPUs</span> (Power Policy Units) — one per core, one per cluster, one per `SYSTOP`, all memory-mapped in the SCP's address space:

```c
/* product/juno/include/system_mmap.h */
#define PPU_BASE              (PERIPHERAL_BASE + 0x20000)   /* 0x44020000 */
#define PPU_BIG_CPU0_BASE     (PPU_BASE + 0x0000)
#define PPU_BIG_SSTOP_BASE    (PPU_BASE + 0x0080)   /* big cluster */
#define PPU_LITTLE_CPU0_BASE  (PPU_BASE + 0x0100)
#define PPU_SYSTOP_BASE       (PPU_BASE + 0x0300)
```

The entire `power_mode_on()` call chain bottoms out in a __single store__ followed by a poll:

```c
/* module/ppu_v1/src/ppu_v1.c */
power_policy = ppu->ppu_reg->PWPR & ~(PPU_V1_PWPR_POLICY | PPU_V1_PWPR_DYNAMIC_EN);
PPU_V1_WRITE_PPU_REG(ppu, PWPR, power_policy | ppu_mode);   /* MODE_ON = 8 */

/* ... then wait for hardware to confirm */
while ((ppu->ppu_reg->PWSR &
        (PPU_V1_PWSR_PWR_STATUS | PPU_V1_PWSR_PWR_DYN_STATUS)) != ppu_mode)
    continue;
```

Write the policy field in <span class="emphasizer_code_function_2">PWPR</span>, poll <span class="emphasizer_code_function_2">PWSR</span> until it reports arrival. The PPU hardware performs the rest itself — power switches, clamps, RAM retention, and __reset release__. Software only states intent.

Order matters: cluster PPU first (L2, interconnect port, cluster clock), then core PPU. Both `juno_rom` and the Total Compute BL1 modules respect that.

### Reaching AP Memory — Two Generations

__Juno__ uses fixed address decode. The Cortex-M3's flat 4 GiB space has chunks hardwired onto the system interconnect:

```c
/* product/juno/include/system_mmap.h */
#define EXTERNAL_DEV_BASE    UINT32_C(0xA0000000)
#define TRUSTED_RAM_BASE     (EXTERNAL_DEV_BASE + 0x04000000)  /* 0xA4000000 */
```

`TRUSTED_RAM_BASE` is _just a pointer_. The byte copy in `mod_bootloader_boot.S` is an ordinary `ldrb` from `0xA4000000` that the address decoder routes out the SCP's AXI master port and across the interconnect. No mailbox, no DMA, no special instruction.

__Modern platforms__ outgrew 32 bits, so an <span class="emphasizer_code_function_2">ATU</span> (Address Translation Unit) windows the SCP's master port. Configuration is a table of regions:

```c
/* module/atu/include/mod_atu.h */
struct FWK_PACKED atu_region_map {
    fwk_id_t region_owner_id;
    uint32_t log_addr_base;   /* 32-bit, SCP space    */
    uint64_t phy_addr_base;   /* 64-bit, system space */
    size_t   region_size;
    uint32_t attributes;      /* Root / Secure / NS / Realm PAS */
};
```

The exact address windows are platform-specific. The ATU configuration maps a 32-bit SCP logical range onto a system physical address, while drivers above it still dereference their configured logical addresses.

The `attributes` field is the security-relevant part. Each window carries a <span class="emphasizer_code_function_2">PAS</span> (Physical Address Space) tag — Root, Secure, Non-Secure or Realm — so the same physical DRAM can be reached through a secure window or a non-secure one, and the interconnect enforces the difference.

Two ownership modes exist, and the difference matters:

- <span class="emphasizer_code_function_2">SCP_ENABLE_ATU_MANAGE</span> — the SCP owns the ATU configuration registers and programs the table directly.
- <span class="emphasizer_code_function_2">SCP_ENABLE_ATU_DELEGATE</span> — the SCP has __no access__ to them. A Root-of-Trust component owns the ATU; the SCP sends translation requests over a transport channel and the RoT decides.

Delegate mode is the direction newer platforms take, and it closes a real hole: it removes the SCP's ability to grant itself arbitrary system access. Both flags can be set at once on platforms with multiple ATU devices.


## Reset Vectors

Here is a distinction that is very easy to get wrong, and I got it wrong myself before reading carefully. Two registers get conflated, and they are __not the same thing__.

| | `RVBAR_EL3` | `RVBARADDR_LW` / `_UP` |
|---|---|---|
| What | AArch64 __system register__ | Memory-mapped register in the CPU __PIK__ |
| Where | Inside the A-core | Outside the core, in SCP peripheral space |
| Reachable by | That core only, via `MRS` | Any bus master that can address it — the SCP |
| Writable | No | Yes |

<br>
`RVBAR_EL3` is inside the core, read-only, reachable only by that core executing `MRS x0, RVBAR_EL3`. It is not memory-mapped anywhere. __The SCP can never touch it.__

What the SCP writes is a different register entirely — a latch in the <span class="emphasizer_code_function_2">CPU PIK</span>, a power and configuration block sitting _outside_ the core, in the SCP's own peripheral address space:

```c
/* product/totalcompute/common/include/cpu_pik.h */
struct static_config_reg {
    FWK_RW uint32_t STATIC_CONFIG;
    FWK_RW uint32_t RVBARADDR_LW;
    FWK_RW uint32_t RVBARADDR_UP;
    uint32_t RESERVED;
};

struct pik_cpu_reg {
    FWK_RW uint32_t CLUSTER_CONFIG;
    uint8_t RESERVED0[0x100 - 0x4];
    struct static_config_reg STATIC_CONFIG[MAX_PIK_SUPPORTED_CPUS];  /* @ 0x100 */
    /* ... */
};
```

One 16-byte slot per core, ten cores per cluster block. That latch's output is __permanently wired__ to the core's `RVBARADDR` input pins. Writing it changes what those wires carry — immediately, continuously. Nobody performs a bus read; the A-core has no idea the PIK exists.

<div style="border:1px solid #ddd; background:#fcfcfc; padding:14px 10px; margin:20px 0; overflow-x:auto;">
<svg viewBox="0 0 720 200" style="width:100%; height:auto; min-width:600px; font-family:'Courier New',Courier,monospace;" role="img" aria-label="PIK latch wired to A-core reset vector input pins">
<rect x="16" y="40" width="196" height="80" fill="#eee" stroke="#b0b0b0"/>
<text x="30" y="60" style="font-size:12px; font-weight:bold; fill:#222;">CPU PIK</text>
<text x="30" y="80" style="font-size:11px; fill:#222;">RVBARADDR_UP</text>
<text x="30" y="97" style="font-size:11px; fill:#222;">RVBARADDR_LW</text>
<text x="30" y="113" style="font-size:9.5px; fill:#666;">SCP writes here</text>
<text x="254" y="74" style="font-size:9.5px; fill:#d9532f;">64 wires, always live</text>
<rect x="398" y="40" width="204" height="118" fill="#eee" stroke="#b0b0b0"/>
<text x="412" y="60" style="font-size:12px; font-weight:bold; fill:#222;">A-core</text>
<text x="412" y="88" style="font-size:11px; fill:#222;">RVBARADDR pins</text>
<text x="412" y="107" style="font-size:9.5px; fill:#666;">at reset release, hardware</text>
<text x="412" y="120" style="font-size:9.5px; fill:#666;">snapshots the wires into...</text>
<rect x="412" y="128" width="120" height="21" fill="#ffeee9" stroke="#fe7e61"/>
<text x="421" y="143" style="font-size:11px; font-weight:bold; fill:#d9532f;">RVBAR_EL3</text>
<path d="M213 80 L 300 80 L 300 92 L 397 92" stroke="#fe7e61" stroke-width="1.5" stroke-linecap="square" stroke-linejoin="miter" fill="none"/>
<path d="M213 96 L 288 96 L 288 102 L 397 102" stroke="#fe7e61" stroke-width="1.5" stroke-linecap="square" stroke-linejoin="miter" fill="none"/>
<path d="M230 180 L 380 180" stroke="#404040" stroke-width="1.3" fill="none" marker-end="url(#a2)"/>
<text x="30" y="184" style="font-size:9.5px; fill:#666;">PPU deasserts reset -&gt;</text>
<text x="420" y="184" style="font-size:9.5px; fill:#666;">core fetches first instruction</text>
<defs><marker id="a2" markerWidth="7" markerHeight="7" refX="6" refY="3.5" orient="auto"><path d="M0 0 L7 3.5 L0 7 z" fill="#404040"/></marker></defs>
</svg>
</div>

The closest everyday analogy is __strapping pins__ or DIP switches on a board: set before power-on, sampled by the chip at reset. Except here the switches are a register the SCP can write.

This is exactly why the write must _precede_ the PPU power-on. Once reset releases, the value is latched, and writing the PIK register afterward does nothing for that core until its next reset.

Juno cannot set an AP entry point at all — not for permission reasons, but because its Cortex-A cores have `RVBAR` tied to a fixed value in silicon. There is no PIK register to write.

### A Misconception Worth Killing

It is tempting to assume that a runtime PSCI `CPU_ON` carries an entry point to the SCP over SCMI, and that the SCP then programs `RVBARADDR` per core.

__It does not.__ The SCMI message is three words and contains no address:

```c
/* module/scmi_power_domain/include/internal/scmi_power_domain.h */
struct scmi_pd_power_state_set_a2p {
    uint32_t flags;
    uint32_t domain_id;
    uint32_t power_state;
};
```

Grepping `entry_point`, `entrypoint` and `entry_addr` across `power_domain/`, `scmi_power_domain/` and `ppu_v1/` returns nothing at all. The SCP is __never told where a core should start__.

### What Actually Happens — A Mailbox

`RVBARADDR` is programmed __once__, at system init, with the same address written to every core in every cluster — TF-A's warm-boot entry. Morello sources it from a board scratch register populated by earlier boot firmware:

```c
/* product/morello/module/morello_system/src/mod_morello_system.c */
for (cluster_idx = 0; cluster_idx < cluster_count; cluster_idx++) {
    for (core_idx = 0; core_idx < morello_core_get_core_per_cluster_count(cluster_idx); core_idx++) {
        PIK_CLUSTER(cluster_idx)->STATIC_CONFIG[core_idx].RVBARADDR_LW = SCC->BOOT_GPR2;
        PIK_CLUSTER(cluster_idx)->STATIC_CONFIG[core_idx].RVBARADDR_UP = SCC->BOOT_GPR3;
    }
}
```

It never changes again. The per-`CPU_ON` entry point travels instead through a __64-byte__ region of shared trusted SRAM whose contents are explicitly none of the SCP's business. From `product/juno/include/software_mmap.h`:

<br>
<hr class="line_1">
_Application Processor Context Area: The usage of this area is defined by the firmware running on the Application Processors. The SCP firmware must zero this memory before releasing any Application Processors._
<hr class="line_1">
<br>
```
 4096 +----------------------------+
      |  AP Context Area      64B  |  <- TF-A's mailbox. SCP zeroes it, never reads it.
      +----------------------------+
      |  Unused              256B  |
      +----------------------------+
      |  SCMI payload  P2A   128B  |
      +----------------------------+
      |  SCMI payload  A2P   128B  |
      +----------------------------+
      |  SDS region         3520B  |  <- the boot handshake from stage 2
    0 +----------------------------+
```

So a PSCI `CPU_ON` resolves as __AP -> shared memory -> AP__:

1. BL31 writes the entry point into the AP context area.
2. BL31 sends SCMI `POWER_STATE_SET` — domain and state only.
3. The SCP powers the domain via the PPU.
4. The core latches the _fixed_ `RVBAR` and runs TF-A's warm-boot code.
5. TF-A reads the mailbox, finds the real entry point, and branches there.

The SCP supplies _power_; TF-A supplies the _destination_. They never exchange the address. That division is deliberate — it keeps the entry point inside the AP's own trust domain, so a compromised SCP cannot redirect where a secure core begins executing.


## SCMI at Runtime

Before the doorbell, BL31 writes the message — a header word plus the three parameter words — into the shared payload area. Then it writes a bit into an <span class="emphasizer_code_function_2">MHU</span> (Message Handling Unit) register, which asserts a physical interrupt line into the SCP's __NVIC__.

```
                                MHU doorbell bit set
                                |
                                v  hardware interrupt line -> NVIC -> vector table
                                +-------------------------------------------------------+
                                | mhu2_isr()                            INTERRUPT CTX   |
                                |   slot = __builtin_ctz(recv_channel->STAT)            |
                                |   bound_channel->driver_input_api->signal_message()   |
                                |   recv_channel->STAT_CLEAR = 1 << slot                |
                                +-------------------------------------------------------+
                                |  transport_signal_message -> scmi's signal_message
                                v
                                fwk_put_event(light event, target = scmi service)
                                |  fwk_is_interrupt_context() == true
                                v  -> pushed onto ctx.isr_event_queue
                                ISR RETURNS.  Total work: ~20 instructions.
                                ---------------------------------------------------------
                                |  main loop: process_isr() migrates to event_queue
                                v
                                +-------------------------------------------------------+
                                | scmi_process_event()                    THREAD CTX    |
                                |   read header word from shared memory                 |
                                |   protocol_id = read_protocol_id(header)    /* 0x11 */|
                                |   message_id  = read_message_id(header)     /* 0x04 */|
                                |   idx = scmi_protocol_id_to_idx[protocol_id]          |
                                |   protocol_table[idx].message_handler(...)            |
                                +-------------------------------------------------------+
                                |
                                v
                                scmi_pd_message_handler -> handler_table[message_id]
                                |
                                v
                                scmi_pd_power_state_set_handler()
                                |- scmi_pd_agent_check()       <- identify the agent
                                |- ..._parameters_check()      <- domain valid, reserved bits zero
                                |- ..._type_check()            <- core / cluster / device
                                \- pd_api->set_state(pd_id, composite_state)
                                |
                                v
                                power_domain HAL -> ppu_v1 -> PWPR = ON, poll PWSR
                                
```

### The ISR Does Almost Nothing

It identifies which slot rang, calls one function pointer, clears the status bit, returns. That call chain ends in `fwk_put_event()`, which detects interrupt context and parks a _light_ event on `isr_event_queue`. All parsing, permission checking and PPU access happens __later__, on the main loop.

This is the framework design paying off. Interrupt latency stays bounded and near-constant regardless of how expensive the message is, and because every handler runs single-threaded on one loop, there is no locking anywhere in the SCMI or power-domain code.

### Dispatch Is Two Table Lookups

`scmi_protocol_id_to_idx[]` maps the wire protocol ID (`0x11` = power domain) to an index; `protocol_table[]` holds each protocol module's `message_handler`. Both are populated during the __bind__ phase by the same API-pointer handshake every module uses.

Each protocol then has its own table keyed by message ID:

```c
/* module/scmi_power_domain/src/mod_scmi_power_domain.c */
static handler_table_t handler_table[MOD_SCMI_PD_POWER_COMMAND_COUNT] = {
    /* ... */
    [MOD_SCMI_PD_POWER_STATE_SET] = scmi_pd_power_state_set_handler,
};
```

A `(protocol_id, message_id)` pair off the wire is indexed twice and lands on a C function. No parsing beyond bitfield extraction from the header word.

### Response and Timing

`respond()` writes the return payload into the __P2A__ area of shared memory, then the transport rings the MHU doorbell in the other direction, interrupting the AP.

For a core power-on the SCMI specification makes `POWER_STATE_SET` __asynchronous__, and the handler forces `is_sync = false` regardless of what the agent's flags requested. The SCP acknowledges immediately and the PPU transition completes afterward. Cluster domains are synchronous; an async request against one is rejected with `SCMI_NOT_SUPPORTED`.

When enabled, the <span class="emphasizer_code_function_2">resource_perms</span> module is consulted by `scmi_pd_permissions_handler()` before the protocol handler runs. `scmi_pd_agent_check()` merely obtains the agent ID. Generic message validation, resource permissions, handler type checks and platform policy together constrain a request before `pd_api->set_state()` reaches hardware.


## Observations

__Release builds skip some event validation.__ Event-ID, target module-index, and notification/response-consistency checks in `__fwk_put_event()` sit inside `#ifdef BUILD_MODE_DEBUG`. Source-ID type and entity validation still runs in production. This is a footprint trade: malformed event metadata receives substantially less validation in release builds.

__Spin-waits in ROM are unbounded.__ The SDS valid-flag loop has no timeout. If AP-side firmware never publishes — bad image, AP hang — the SCP wedges in ROM indefinitely. Juno reloads the watchdog immediately before `load_image()` and so recovers via reset; the Total Compute BL1 path has no equivalent guard visible in the module.

__The alloc-only rule is not enforced by the API.__ The coding rules mandate allocate-only memory with no free or realloc. But `fwk_mm.c` exports `fwk_mm_free()` and `fwk_mm_realloc()` as thin wrappers around `free` and `realloc`. The rule lives in review, not in the interface — and with the repository now read-only, that review gate is gone.

__The event pool is a fixed global.__ One misbehaving module leaking events starves the whole system, and `fwk_unexpected()` is the only signal.

__Init order is hand-maintained.__ Each firmware target has a module list, a topological sort kept correct by hand, with a comment warning that reordering breaks initialization.

__`interface/` is underused.__ Four header-only contracts exist for 110 modules. Most cross-module coupling still goes directly through `mod_X.h` API structs.

## References

### Arm Specifications

- [Power Control System Architecture (DEN0050C)](https://developer.arm.com/documentation/den0050/d/?lang=en)
- [System Control and Management Interface (DEN0056A)](https://developer.arm.com/documentation/den0056/d/)
- [Arm v7-M Architecture Reference Manual](https://developer.arm.com/documentation/ddi0403/latest/)
- [Arm v8-M Architecture Reference Manual](https://developer.arm.com/documentation/ddi0553/latest/)

### Source

- [SCP-firmware source snapshot (`b34e5ce`)](https://github.com/ARM-software/SCP-firmware/tree/b34e5ce) — upstream version 2.14.0 reviewed here
- [Trusted Firmware-A](https://github.com/ARM-software/arm-trusted-firmware) — the other side of every handshake in this post
- [TF-A SCMI / SCP interface documentation](https://trustedfirmware-a.readthedocs.io/en/latest/design/index.html)

### In-tree Documentation

- `doc/framework.md` — framework component model
- `doc/deferred_response_architecture.md` — the `FWK_PENDING` flow
- `doc/scp_firmware_threat_model.md` — STRIDE analysis, trust boundary at the SCMI agent
- `doc/code_rules.md` — coding rules, including the allocate-only memory policy
- `module/atu/doc/` — ATU manage vs delegate modes

### Related

- [Linux SCMI client implementation](https://github.com/torvalds/linux/tree/master/drivers/firmware/arm_scmi)
- [PSCI specification (DEN0022)](https://developer.arm.com/documentation/den0022/latest/) — what `CPU_ON` means on the AP side
