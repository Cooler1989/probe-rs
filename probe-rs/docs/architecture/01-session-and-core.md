# 01 — Session and Core

Sources: [session.rs](../../src/session.rs), [core.rs](../../src/core.rs),
[core/core_state.rs](../../src/core/core_state.rs), [core/registers.rs](../../src/core/registers.rs),
[core/core_status.rs](../../src/core/core_status.rs), [memory/mod.rs](../../src/memory/mod.rs).

## 1. Responsibilities

| Type | Lifetime | Role |
|---|---|---|
| `Session` | Long-lived; the only handle users should store | Owns the `Target`, the architecture interface (which owns the `Probe`) and per-core state. Can be shared across threads as `Arc<FairMutex<Session>>`. |
| `SessionConfig` | Transient | Speed, `WireProtocol`, `Permissions` for `Session::auto_attach`. |
| `Permissions` | Copied into session | Gate for irreversible operations (`erase_all`). |
| `CombinedCoreState` (`pub(crate)`) | Lives inside `Session.cores` | Per-core persistent state: id, resolved access options + debug sequence, architecture-specific cached state. |
| `Core<'probe>` | Short-lived borrow of `Session` | Public façade around `Box<dyn CoreInterface + 'probe>`. Recreated by every `Session::core(i)` call. |
| `CoreInterface` | Trait | Architecture-neutral core control API; extends `MemoryInterface`. |

## 2. Static structure

```mermaid
classDiagram
    direction LR

    class Session {
        -target: Target
        -interfaces: ArchitectureInterface
        -cores: Vec~CombinedCoreState~
        -configured_trace_sink: Option~TraceSink~
        +auto_attach(target, SessionConfig) Session
        +core(index) Core
        +list_cores() Vec
        +halted_access(f)
        +target() Target
        +get_arm_interface()
        +get_riscv_interface(core)
        +get_xtensa_interface(core)
        +sequence_erase_all()
        +setup_tracing() / swo_reader()
        +resume_all_cores()
        +clear_all_hw_breakpoints()
    }

    class ArchitectureInterface {
        <<enum, private>>
        Arm(Box dyn ArmDebugInterface)
        ArmWithRiscv(arm, riscv_mem_ap_cores)
        Jtag(Probe, Vec~JtagInterface~)
        +attach(target, CombinedCoreState) Core
    }

    class JtagInterface {
        <<enum, private>>
        Riscv(RiscvDebugInterfaceState)
        Xtensa(XtensaDebugInterfaceState)
        Unknown
    }

    class CombinedCoreState {
        +core_state: CoreState
        +specific_state: SpecificCoreState
        +id: usize
        +attach_arm(target, iface) Core
        +attach_riscv(target, iface) Core
        +attach_xtensa(target, iface) Core
        +enable_arm_debug(iface)
        +arm_reset_catch_set(iface)
        +jtag_tap_index() usize
        +arm_memory_ap()
    }

    class CoreState {
        -core_access_options: ResolvedCoreOptions
        +is_arm() bool
        +memory_ap()
    }

    class ResolvedCoreOptions {
        <<enum>>
        Arm(Arc dyn ArmDebugSequence, ArmCoreAccessOptions)
        Riscv(Arc dyn RiscvDebugSequence, RiscvCoreAccessOptions)
        Xtensa(Arc dyn XtensaDebugSequence, XtensaCoreAccessOptions)
    }

    class SpecificCoreState {
        <<enum>>
        Armv6m(CortexMState)
        Armv7m / Armv7em / Armv8m(CortexMState)
        Armv7a / Armv7r / Armv8a(CortexARState)
        Riscv / Riscv64(RiscvCoreState)
        Xtensa(XtensaCoreState)
    }

    class Core {
        -id: usize
        -name: str
        -target: Target
        -inner: Box dyn CoreInterface
        +halt() / run() / step()
        +reset() / reset_and_halt()
        +read_core_reg() / write_core_reg()
        +set_hw_breakpoint() / clear_hw_breakpoint()
        +memory_regions()
    }

    class CoreInterface {
        <<trait>>
    }
    class MemoryInterface {
        <<trait>>
    }

    Session *-- ArchitectureInterface
    Session *-- "1..*" CombinedCoreState
    ArchitectureInterface o-- JtagInterface
    CombinedCoreState *-- CoreState
    CombinedCoreState *-- SpecificCoreState
    CoreState *-- ResolvedCoreOptions
    Session ..> Core : creates via core(i)
    Core o-- CoreInterface
    CoreInterface --|> MemoryInterface
```

### `CombinedCoreState` in detail

`CombinedCoreState` (in [core_state.rs](../../src/core/core_state.rs)) is the glue
between persistent per-core data and the transient `Core` handle:

* **`core_state.core_access_options: ResolvedCoreOptions`** — created by
  `ResolvedCoreOptions::new(target, options)` in [core.rs](../../src/core.rs). It pairs
  the YAML core access options (memory AP / debug base / CTI base, hart id, JTAG TAP)
  with the debug sequence `Arc`. If the target's `debug_sequence` doesn't match the
  core's architecture (e.g. RP235x RISC-V cores on an ARM-sequenced family) the
  default sequence for that architecture is used.
* **`specific_state: SpecificCoreState`** — architecture-specific cache that must
  outlive a `Core` borrow, for example:
  * `CortexMState`: `initialized`, `hw_breakpoints_enabled`, `current_state: CoreStatus`,
    `fp_present`, pending semihosting command
  * `CortexARState`: `is_64_bit`, `register_cache`, `fp_reg_count`, ...
  * `RiscvCoreState`: breakpoint state, `pc_written`, `fp_present`, ...
  * `XtensaCoreState`: `enabled`, `breakpoint_set`, `pc_written`, ...
* **`id`** — index into `Target.cores`.

The `attach_*` methods consume the architecture communication interface plus
`&mut specific_state` and construct the concrete core driver
(`Armv6m`, `Armv7m`, `Armv8m`, `Armv7ar`, `Armv8a`, `Riscv32`, `Riscv64`, `Xtensa`),
boxing it into a `Core`.

## 3. Creating a session

`Probe::attach(...)` → `Session::new(probe, TargetSelector, AttachMethod, Permissions, &Registry)`.

```mermaid
flowchart TD
    A["Session::new"] --> B["get_target_from_selector"]
    B -->|"Unspecified(name)"| B1["registry.get_target_by_name"]
    B -->|"Specified(target)"| B2["use as-is"]
    B -->|"Auto"| B3["probe.attach_to_unspecified<br/>vendor::auto_determine_target"]
    B1 & B2 & B3 --> C["for each target core:<br/>Core::create_state → CombinedCoreState"]
    C --> D{"default core has<br/>memory AP?"}
    D -- "yes (ARM DAP, or RISC-V over mem-AP)" --> E["attach_arm_debug_interface"]
    D -- "no (JTAG RISC-V / Xtensa)" --> F["attach_jtag"]
    E --> G["session.clear_all_hw_breakpoints()"]
    F --> G
    G --> H["Session"]
```

### 3.1 ARM / DAP path (`attach_arm_debug_interface`)

```mermaid
sequenceDiagram
    autonumber
    participant S as Session
    participant P as Probe
    participant Seq as ArmDebugSequence
    participant I as ArmDebugInterface
    participant CS as CombinedCoreState[i]

    opt AttachMethod::UnderReset
        S->>Seq: reset_hardware_assert(dap_probe)
        Note right of S: falls back to probe.target_reset_assert()
    end
    S->>P: set_expected_scan_chain (if JTAG chain in YAML)
    S->>P: attach_to_unspecified()
    S->>P: try_into_arm_debug_interface(sequence)
    P-->>S: Box dyn ArmDebugInterface
    S->>I: select_debug_port(default_dp)
    S->>Seq: debug_device_unlock(iface, default_ap, permissions)
    alt ReAttachRequired
        S->>S: reattach_arm_interface
    end
    alt UnderReset
        loop every ARM core
            S->>CS: arm_reset_catch_set(iface)
        end
        S->>Seq: reset_hardware_deassert
        loop every ARM core
            S->>CS: enable_arm_debug → Seq.debug_core_start
        end
        S->>S: build_arm_interfaces → Arm or ArmWithRiscv
        loop every ARM core
            S->>S: core(i).wait_for_core_halted + reset_catch_clear
        end
    else Normal
        loop every ARM core
            S->>CS: enable_arm_debug → Seq.debug_core_start
        end
        S->>S: build_arm_interfaces
    end
```

### 3.2 JTAG path (`attach_jtag`) — RISC-V and Xtensa

1. Apply the YAML scan chain (`set_scan_chain` when `force_scan_chain`, otherwise
   `set_expected_scan_chain`), then `probe.attach_to_unspecified()`.
2. Determine the TAP count (from scan chain, or highest `jtag_tap_index` + 1).
3. For every core, initialise its TAP slot once:
   * RISC-V → `probe.try_get_riscv_interface_builder()`, `create_state()`,
     `attach_auto(...)`, `enter_debug_mode()` → `JtagInterface::Riscv(state)`
   * Xtensa → `JtagInterface::Xtensa(XtensaDebugInterfaceState::default())`
   * Mixing architectures on the same TAP is rejected.
4. Build `ArchitectureInterface::Jtag(probe, interfaces)`.
5. For RISC-V targets call `RiscvDebugSequence::on_connect` for every core.

## 4. Obtaining a `Core`

`Session::core(i)` borrows `self` mutably and calls
`ArchitectureInterface::attach(&target, &mut cores[i])`:

```mermaid
flowchart LR
    SC["Session::core(i)"] --> AI{"ArchitectureInterface"}
    AI -- "Arm(iface)" --> A1["CombinedCoreState::attach_arm<br/>seq.on_attach → iface.memory_interface(ap)<br/>→ Armv6m / Armv7m / Armv8m / Armv7ar / Armv8a"]
    AI -- "ArmWithRiscv" --> A2{"RISC-V core<br/>on mem-AP?"}
    A2 -- yes --> A3["arm.memory_interface(ap) → MemApDtm<br/>→ RiscvCommunicationInterface<br/>→ attach_riscv"]
    A2 -- no --> A1
    AI -- "Jtag(probe, ifaces)" --> J1["probe.select_target(tap)"]
    J1 --> J2{"ifaces[tap]"}
    J2 -- Riscv --> J3["riscv builder.attach_auto → attach_riscv<br/>(select_hart) → Riscv32 / Riscv64"]
    J2 -- Xtensa --> J4["probe.try_get_xtensa_interface → attach_xtensa → Xtensa"]
    A1 & A3 & J3 & J4 --> CORE["Core (boxed CoreInterface)"]
```

Errors `ArmError::CoreDisabled`, `XtensaError::CoreDisabled` and
`RiscvError::HartUnavailable` are mapped to `Error::CoreDisabled(i)` so callers
can retry later.

> The docs on `Session::core` stress: *never store the `Core` handle for long*.
> All persistent state lives in the `Session`; the first call runs init sequences,
> subsequent calls are cheap.

## 5. `CoreInterface` contract

`CoreInterface: MemoryInterface` ([core.rs](../../src/core.rs)) — implemented by
`Armv6m`, `Armv7m`, `Armv8m`, `Armv7ar`, `Armv8a`, `Riscv32`/`Riscv64`
(`RiscvCore`), `Xtensa`, and by `Core` itself.

| Group | Methods |
|---|---|
| Run control | `wait_for_core_halted`, `core_halted`, `status`, `halt`, `run`, `reset`, `reset_and_halt`, `step` |
| Registers | `read_core_reg`, `write_core_reg`, `registers`, `program_counter`, `frame_pointer`, `stack_pointer`, `return_address` |
| Breakpoints | `available_breakpoint_units`, `hw_breakpoints`, `enable_breakpoints`, `set_hw_breakpoint`, `clear_hw_breakpoint`, `hw_breakpoints_enabled` |
| Introspection | `architecture`, `core_type`, `instruction_set`, `endianness`, `fpu_support`, `floating_point_register_count`, `is_64_bit` |
| Debug events | `reset_catch_set`, `reset_catch_clear`, `enable_vector_catch`, `disable_vector_catch`, `debug_core_stop`, `spill_registers` |

`MemoryInterface` ([memory/mod.rs](../../src/memory/mod.rs)) provides
`read_word_{8,16,32,64}`, `read_{8,16,32,64}`, `read`, the write equivalents,
`supports_native_64bit_access`, `supports_8bit_transfers`, `flush`, and batched
`execute_memory_operations`. `Core` gets it via the `CoreMemoryInterface` adapter.

## 6. Core status model

```mermaid
stateDiagram-v2
    [*] --> Unknown
    Unknown --> Running : status()
    Unknown --> Halted : status()
    Running --> Halted : halt() / breakpoint / step done / exception catch
    Halted --> Running : run()
    Halted --> Halted : step()
    Running --> Halted : reset_and_halt()
    Halted --> Running : reset()
    Running --> Sleeping : WFI / WFE
    Sleeping --> Running
    Running --> LockedUp : fault lockup (Cortex-M)
    LockedUp --> Halted : reset_and_halt()
```

`CoreStatus::Halted(HaltReason)` carries the reason (`Breakpoint(BreakpointCause)`,
`Multiple`, `Exception`, `Watchpoint`, `Step`, `Request`, `External`, `Unknown`). A semihosting
request is reported as a breakpoint cause, see
[06-rtt-semihosting-trace.md](./06-rtt-semihosting-trace.md).

## 7. Concurrency model

* `Session` is `Send` (asserted at compile time in `session.rs`).
* Multi-stakeholder use (GDB + RTT + flashing in `probe-rs-tools`) wraps the
  session in a mutex; each stakeholder acquires it, calls `core(i)`, performs an
  atomic operation and drops the borrow.
* `Session::halted_access(f)` halts all running cores, executes `f`, then resumes
  the cores that were running before.
* `Drop for Session` clears all hardware breakpoints and calls `debug_core_stop()`
  on every enabled core (disabled cores are skipped); failures are only logged.
