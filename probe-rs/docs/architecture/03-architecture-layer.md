# 03 — Architecture layer (ARM, RISC-V, Xtensa)

Sources: [architecture/arm/](../../src/architecture/arm/mod.rs),
[architecture/riscv/](../../src/architecture/riscv/mod.rs),
[architecture/xtensa/](../../src/architecture/xtensa/mod.rs).

The architecture layer turns raw probe capabilities into **core drivers** that
implement `CoreInterface`. Each architecture has:

| Concern | ARM | RISC-V | Xtensa |
|---|---|---|---|
| Transport to debug logic | `ArmDebugInterface` (DP/AP) | `DtmAccess` (JTAG DTM, tunnelled JTAG, mem-AP DTM) | `Xdm` over JTAG (NAR/NDR) |
| Communication interface | `ArmCommunicationInterface` or probe-specific impl | `RiscvCommunicationInterface<'state>` | `XtensaCommunicationInterface<'probe>` |
| Persistent interface state | `DpState` map inside interface | `RiscvDebugInterfaceState` | `XtensaDebugInterfaceState` |
| Core drivers | `Armv6m`, `Armv7m`, `Armv8m`, `Armv7ar`, `Armv8a` | `RiscvCore<X: XlenMode>` = `Riscv32` / `Riscv64` | `Xtensa` |
| Per-core cache | `CortexMState`, `CortexARState` | `RiscvCoreState` | `XtensaCoreState` |
| Chip customisation hook | `ArmDebugSequence` | `RiscvDebugSequence` | `XtensaDebugSequence` |
| Error type | `ArmError` | `RiscvError` | `XtensaError` |

```mermaid
flowchart TB
    subgraph Core["CoreInterface implementors"]
        M6["Armv6m"]; M7["Armv7m"]; M8["Armv8m"]; AR["Armv7ar"]; A8["Armv8a"]
        RV["RiscvCore (Riscv32 / Riscv64)"]
        XT["Xtensa"]
    end

    subgraph ARM["arm"]
        MI["ArmMemoryInterface<br/>(ADIMemoryInterface over a MemoryAp)"]
        ADI["ArmDebugInterface<br/>(ArmCommunicationInterface / StlinkArmDebug / ...)"]
        DAP["DapAccess → RawDapAccess"]
    end

    subgraph RISCV["riscv"]
        RCI["RiscvCommunicationInterface"]
        DTM["DtmAccess: JtagDtm · TunneledJtagDtm · MemApDtm"]
    end

    subgraph XTENSA["xtensa"]
        XCI["XtensaCommunicationInterface"]
        XDM["Xdm (NAR / NDR)"]
    end

    PROBE["Probe / DebugProbe / JtagAccess"]

    M6 & M7 & M8 & AR & A8 --> MI --> ADI --> DAP --> PROBE
    RV --> RCI --> DTM
    DTM -->|JTAG| PROBE
    DTM -->|"mem-AP (RP2350)"| MI
    XT --> XCI --> XDM --> PROBE
```

---

## 1. ARM (ADIv5 / ADIv6, CoreSight)

### 1.1 Layering

```mermaid
classDiagram
    direction TB

    class RawDapAccess {
        <<trait>>
        raw_read_register / raw_write_register
        raw_read_block / raw_write_block
        swj_sequence / swj_pins / jtag_sequence
        configure_jtag / raw_flush
    }
    class DapAccess {
        <<trait>>
        read_raw_dp_register / write_raw_dp_register
        read_raw_ap_register / write_raw_ap_register
        _repeated variants / flush
        try_dap_probe
    }
    class ArmDebugInterface {
        <<trait>>
        reinitialize()
        access_ports(dp)
        select_debug_port(dp) / current_debug_port()
        memory_interface(ap) Box dyn ArmMemoryInterface
        close() Probe
    }
    class ArmCommunicationInterface {
        -probe: Option Box dyn DapProbe
        -current_dp: Option DpAddress
        -dps: HashMap DpAddress DpState
        -sequence: Arc dyn ArmDebugSequence
    }
    class StlinkArmDebug
    class BlackMagicProbeArmDebug
    class SifliUartArmDebug
    class ArmMemoryInterface {
        <<trait>>
        base_address() / fully_qualified_address()
        get_arm_debug_interface()
        generic_status() / update_core_status()
    }
    class ADIMemoryInterface {
        <<crate-private>>
    }
    class MemoryAp {
        AHB3 · AHB5 · AHB5-HPROT · APB2/3 · APB4/5 · AXI3/4 · AXI5
    }

    ArmDebugInterface --|> DapAccess
    ArmCommunicationInterface ..|> ArmDebugInterface
    StlinkArmDebug ..|> ArmDebugInterface
    BlackMagicProbeArmDebug ..|> ArmDebugInterface
    SifliUartArmDebug ..|> ArmDebugInterface
    ArmCommunicationInterface --> RawDapAccess : uses DapProbe
    ADIMemoryInterface ..|> ArmMemoryInterface
    ADIMemoryInterface --> MemoryAp
    ADIMemoryInterface --> ArmDebugInterface
```

### 1.2 Addressing

* `DpAddress` — `Default` or `Multidrop(targetsel)` (SWD multi-drop, e.g. RP2040).
* `ApAddress` — `V1(u8)` (ADIv5 APSEL) or `V2(ApV2Address)` (ADIv6 hierarchical
  address; root memory interface in [ap/v2/](../../src/architecture/arm/ap/v2/mod.rs)).
* `FullyQualifiedApAddress { dp, ap }` — what the YAML core description and
  `CombinedCoreState::arm_memory_ap()` resolve to.
* DP registers (`Ctrl`, `Abort`, `SelectV1/V3`, `DPIDR`, `TARGETID`, `BASEPTR0/1`, ...)
  and AP registers are strongly typed via `DpRegister` / `ApRegister` traits and
  `bitfield!` macros.

### 1.3 Discovery: ROM tables and CoreSight components

[memory/romtable.rs](../../src/architecture/arm/memory/romtable.rs) walks ROM tables
(`RomTable`, `RomTableEntry`, `ComponentId`, `PeripheralID`) to build a
`CoresightComponent` tree. `read_chip_info_from_rom_table` produces `ArmChipInfo`
(JEP106 manufacturer + part) used for **auto-detection** (see
[04-targets-vendors-plugins.md](./04-targets-vendors-plugins.md)).

[component/](../../src/architecture/arm/component/mod.rs) implements drivers for
DWT, ITM, TPIU, SWO, TMC (ETB/ETF), trace funnel and SCS, used by
`Session::setup_tracing`.

### 1.4 Core drivers

| Driver | File | Cores | Mechanism |
|---|---|---|---|
| `Armv6m` | [core/armv6m.rs](../../src/architecture/arm/core/armv6m.rs) | Cortex-M0/M0+/M1 | DHCSR/DCRSR/DCRDR, BPU |
| `Armv7m` | [core/armv7m.rs](../../src/architecture/arm/core/armv7m.rs) | Cortex-M3/M4/M7 (v7-M, v7E-M) | DHCSR, FPB, DEMCR vector catch |
| `Armv8m` | [core/armv8m.rs](../../src/architecture/arm/core/armv8m.rs) | Cortex-M23/M33/M55/M85 | v8-M debug registers (DHCSR, FPB, DEMCR) |
| `Armv7ar` | [core/armv7ar.rs](../../src/architecture/arm/core/armv7ar.rs) | Cortex-A/R v7 | memory-mapped debug regs at `debug_base`, instruction injection via ITR |
| `Armv8a` | [core/armv8a.rs](../../src/architecture/arm/core/armv8a.rs) | Cortex-A v8 (AArch32/AArch64) | debug regs + **CTI** (`cti_base`) for halt/restart, instruction injection |

Shared helpers live in [core/cortex_m.rs](../../src/architecture/arm/core/cortex_m.rs),
[core/instructions.rs](../../src/architecture/arm/core/instructions.rs) and the
register definitions under [core/registers/](../../src/architecture/arm/core/registers).

### 1.5 `ArmDebugSequence` — CMSIS debug-description hooks

[arm/sequences.rs](../../src/architecture/arm/sequences.rs) mirrors the
Open-CMSIS-Pack debug description. Every method has a spec-compliant default
(`DefaultArmSequence`); vendors override only what they need.

```mermaid
flowchart LR
    subgraph Connect
        RHA["reset_hardware_assert"] --> DPS["debug_port_setup"] --> DPST["debug_port_start"] --> UNL["debug_device_unlock"]
        UNL --> DCS["debug_core_start"] --> OA["on_attach"]
    end
    subgraph Reset
        RCS["reset_catch_set"] --> RS["reset_system"] --> RCC["reset_catch_clear"]
        RHD["reset_hardware_deassert"]
    end
    subgraph Misc
        TS["trace_start"]
        PRR["prepare_running_on_ram"]
        DES["debug_erase_sequence"]
        RSS["recover_support_start"]
        AAP["allowed_access_ports"]
        DPC["debug_port_connect"]
    end
    subgraph Disconnect
        DCSt["debug_core_stop"] --> DPSt["debug_port_stop"]
    end
```

---

## 2. RISC-V (Debug Spec 0.13 / 1.0)

```mermaid
classDiagram
    direction LR
    class RiscvInterfaceBuilder {
        <<trait>>
        create_state() RiscvDebugInterfaceState
        attach(state) RiscvCommunicationInterface
        attach_auto(target, state)
    }
    class DtmAccess {
        <<trait>>
        init() / read_idcode()
        schedule_read / schedule_write / execute
        read_with_timeout / write_with_timeout
        target_reset_assert / _deassert
        clear_error_state()
    }
    class JtagDtm
    class TunneledJtagDtm
    class MemApDtm
    class RiscvCommunicationInterface {
        -dtm: Box dyn DtmAccess
        -state: mut RiscvCommunicationInterfaceState
        +select_hart(n)
        +enter_debug_mode()
        +abstract commands / progbuf / system bus access
    }
    class RiscvCore~X~ {
        Riscv32 = RiscvCore~Xlen32~
        Riscv64 = RiscvCore~Xlen64~
    }
    class RiscvDebugSequence {
        <<trait>>
        on_connect / on_halt
        reset_catch_set / reset_catch_clear
        reset_system_and_halt
        on_unknown_semihosting_command
        prepare_running_on_ram
    }

    JtagDtm ..|> DtmAccess
    TunneledJtagDtm ..|> DtmAccess
    MemApDtm ..|> DtmAccess
    RiscvInterfaceBuilder ..> RiscvCommunicationInterface : creates
    RiscvCommunicationInterface --> DtmAccess
    RiscvCore --> RiscvCommunicationInterface
    RiscvCore --> RiscvDebugSequence
```

* **DTM transports** ([dtm/](../../src/architecture/riscv/dtm/mod.rs)):
  * `JtagDtm` — standard JTAG DTM (`dtmcs`, `dmi`) using batched JTAG queues;
    `JtagDtmBuilder` implements `RiscvInterfaceBuilder` for any `JtagAccess` probe.
  * `TunneledJtagDtm` — DTM reached through a BSCAN tunnel in another TAP (SiFive scheme,
    equivalent to OpenOCD's `riscv use_bscan_tunnel`), e.g. RISC-V soft cores in FPGAs.
  * `MemApDtm` — DM mapped behind an ARM MEM-AP (RP2350 Hazard3 cores); used by the
    `ArchitectureInterface::ArmWithRiscv` variant.
* **Memory access** — `MemoryAccessMethod` / `RiscvBusAccess` choose between
  program buffer, abstract memory commands or the system bus (`sbcs`, `sbaddress*`,
  `sbdata*`), cached per access width in `MemoryAccessConfig`.
* **Harts** — one DM may expose several harts; `CombinedCoreState::attach_riscv`
  calls `select_hart(options.hart_id)`.

## 3. Xtensa (ESP32 family etc.)

```mermaid
flowchart LR
    X["Xtensa (CoreInterface)"] --> XCI["XtensaCommunicationInterface<br/>register cache, windowed regs,<br/>instruction execution via DDR/DIR"]
    XCI --> XDM["Xdm<br/>NAR/NDR access, PowerControl,<br/>PowerStatus, DebugStatus, DebugControl"]
    XDM --> JTAG["JtagAccess (probe)"]
    X --> XS["XtensaDebugSequence<br/>on_connect / on_halt /<br/>reset_system_and_halt / ..."]
    XCI --> XST["XtensaDebugInterfaceState<br/>(persistent in Session)"]
```

* [xdm.rs](../../src/architecture/xtensa/xdm.rs) speaks the Xtensa Debug Module
  over JTAG (Nexus registers, power control).
* [communication_interface.rs](../../src/architecture/xtensa/communication_interface.rs)
  implements halt/resume, register access by injecting instructions
  ([arch/instruction](../../src/architecture/xtensa/arch)), memory access,
  breakpoints (`IBreakEn`), single-step (`ICount`).
* [register_cache.rs](../../src/architecture/xtensa/register_cache.rs) caches
  registers while halted; windowed registers are spilled with `spill_registers`.
* Concrete ESP sequences live in the `probe-rs-espressif` plugin crate.

## 4. Debug sequence selection

```mermaid
flowchart TD
    Y["Target YAML<br/>(chip, family, cores)"] --> V["vendor::try_create_debug_sequence(chip)"]
    V -->|"Some(seq)"| DS["Target.debug_sequence: DebugSequence"]
    V -->|"None"| DEF["DefaultArmSequence / DefaultRiscvSequence /<br/>DefaultXtensaSequence (by core arch)"]
    DEF --> DS
    DS --> RCO["ResolvedCoreOptions::new<br/>(per core, falls back to default<br/>if architecture mismatches)"]
    RCO --> CCS["CombinedCoreState.core_state"]
```

`DebugSequence` is `Arm(Arc<dyn ArmDebugSequence>)`,
`Riscv(Arc<dyn RiscvDebugSequence>)` or `Xtensa(Arc<dyn XtensaDebugSequence>)`
([config/target.rs](../../src/config/target.rs)).
