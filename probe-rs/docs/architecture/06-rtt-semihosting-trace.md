# 06 — RTT, semihosting, tracing and core dumps

Sources: [rtt.rs](../../src/rtt.rs), [rtt/channel.rs](../../src/rtt/channel.rs),
[semihosting.rs](../../src/semihosting.rs),
[architecture/arm/component/](../../src/architecture/arm/component/mod.rs),
[architecture/arm/swo/](../../src/architecture/arm/swo/mod.rs),
[core/dump.rs](../../src/core/dump.rs).

These features are built **on top of** the `Core` / `Session` API — they only use
memory access, register access and run control, so they work for every
architecture unless noted otherwise.

```mermaid
flowchart TB
    subgraph Features
        RTT["RTT<br/>Rtt, UpChannel, DownChannel"]
        SH["Semihosting<br/>SemihostingCommand"]
        TR["SWO / ITM / DWT tracing<br/>(ARM only)"]
        CD["CoreDump<br/>(feature coredump)"]
    end
    C["Core (MemoryInterface + CoreInterface)"]
    S["Session"]
    RTT --> C
    SH --> C
    CD --> C
    TR --> S
    S --> AI["ArmDebugInterface + CoreSight components"]
```

## 1. RTT (SEGGER Real-Time Transfer)

RTT uses a control block in target RAM (magic `"SEGGER RTT"`) containing ring
buffers. The host polls/updates the read/write pointers through the debug
interface while the target keeps running.

```mermaid
classDiagram
    class Rtt {
        -ptr: u64
        -up_channels: Vec~UpChannel~
        -down_channels: Vec~DownChannel~
        +attach(core) Rtt
        +attach_region(core, ScanRegion) Rtt
        +attach_at(core, addr) Rtt
        +find_control_block(core, ScanRegion) u64
        +clear_control_block(core, ScanRegion)
        +up_channels() / down_channels()
        +up_channel(n) / down_channel(n)
    }
    class ScanRegion {
        <<enum>>
        Ram
        Ranges(Vec of Range)
        Exact(u64)
    }
    class Channel {
        <<crate-private>>
        number, name, metadata_ptr, info
        read_buffer_offsets()
        write_write_buffer_ptr()
        mode() / set_mode()
    }
    class UpChannel {
        +read(core, buf) usize
        +number() / name() / buffer_size()
        +mode() / set_mode()
    }
    class DownChannel {
        +write(core, buf) usize
    }
    class ChannelMode {
        <<enum>>
        NoBlockSkip
        NoBlockTrim
        BlockIfFull
    }
    Rtt *-- UpChannel
    Rtt *-- DownChannel
    UpChannel *-- Channel
    DownChannel *-- Channel
    Channel ..> ChannelMode
    Rtt ..> ScanRegion
```

```mermaid
sequenceDiagram
    participant H as Host (tool)
    participant C as Core
    participant RAM as Target RAM

    H->>H: control block address from ELF symbol _SEGGER_RTT<br/>(find_rtt_control_block_in_file) or Target.rtt_scan_regions
    H->>C: Rtt::attach_region(core, ScanRegion)
    alt ScanRegion::Exact
        C->>RAM: read header at address
    else Ram / Ranges
        C->>RAM: read regions, search for "SEGGER RTT"
    end
    C-->>H: Rtt (channels parsed)
    loop polling
        H->>C: up_channel.read(core, buf)
        C->>RAM: read WrOff/RdOff, copy data, write RdOff
        H->>C: down_channel.write(core, data)
        C->>RAM: copy data, update WrOff
    end
```

`try_attach_to_rtt` / `try_attach_to_rtt_shared` retry attachment with a timeout
(the control block may only be initialised after the firmware starts). The flash
loader re-uses RTT to stream log output from flash algorithms
(see [05-flashing.md](./05-flashing.md)).

## 2. Semihosting

When target code executes a semihosting trap (`BKPT 0xAB` on ARM, the
`slli x0,x0,0x1f / ebreak / srai x0,x0,7` sequence on RISC-V, `BREAK 1,14` on Xtensa), the core halts.
The core driver detects this and reports
`CoreStatus::Halted(HaltReason::Breakpoint(BreakpointCause::Semihosting(cmd)))`.

```mermaid
flowchart LR
    T["Target executes<br/>semihosting trap"] --> H["Core halts"]
    H --> D["core driver: decode_semihosting_syscall<br/>(op in r0/a0, param block in r1/a1)"]
    D --> CMD["SemihostingCommand"]
    CMD --> APP["Host tool handles it<br/>(print, file I/O, exit)"]
    APP --> R["write result back<br/>(write_status / success / respond_with_handle)"]
    R --> RUN["core.run()"]
```

Supported `SemihostingCommand` variants: `ExitSuccess`, `ExitError`,
`GetCommandLine`, `Open`, `Close`, `WriteConsole`, `Write`, `Read`, `Seek`,
`FileLength`, `Remove`, `Rename`, `Time`, `Errno`, and `Unknown` (forwarded to
`*DebugSequence::on_unknown_semihosting_command` so vendors can handle custom ops).

## 3. ARM tracing (SWO / ITM / DWT / TPIU / ETB)

```mermaid
flowchart LR
    S["Session::setup_tracing(core, TraceSink)"] --> E["component::enable_tracing(core)<br/>(DEMCR.TRCENA)"]
    S --> CO["get_arm_components(dp)<br/>(ROM table walk)"]
    S --> SEQ["ArmDebugSequence::trace_start"]
    S --> SINK{"TraceSink"}
    SINK -- "Swo(SwoConfig)" --> SWO["TPIU/SWO in UART/Manchester mode<br/>+ probe SwoAccess::enable_swo"]
    SINK -- "Tpiu(SwoConfig)" --> TPIU["TPIU parallel port"]
    SINK -- "TraceMemory" --> TMC["TMC / ETB / ETF buffer"]
    SWO --> RD["Session::read_trace_data()<br/>Session::swo_reader() (impl io::Read)"]
    TMC --> RD
    DWT["add_swv_data_trace(unit, addr)<br/>remove_swv_data_trace(unit)"] --> S
```

Components live in [component/](../../src/architecture/arm/component/mod.rs):
`Dwt`, `Itm`, `Tpiu`, `Swo`, `TraceMemoryController` (TMC), `TraceFunnel`, `Scs`.

## 4. Core dumps (feature `coredump`)

`CoreDump::dump_core(&mut core, ranges)` captures:

* all registers (`HashMap<RegisterId, RegisterValue>`),
* memory ranges (`Vec<(Range<u64>, Vec<u8>)>`),
* `instruction_set`, `core_type`, `fpu_support`, FP register count,
  64-bit access support.

`store(path)` / `load(path)` / `load_raw(bytes)` serialise it with MessagePack
(`rmp-serde`). A loaded `CoreDump` can act as a read-only core for
`probe-rs-debug` (stack traces / variable inspection offline).
