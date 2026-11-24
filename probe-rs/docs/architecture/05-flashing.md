# 05 — Flashing

Sources: [flashing/](../../src/flashing/mod.rs) — [download.rs](../../src/flashing/download.rs),
[loader.rs](../../src/flashing/loader.rs), [builder.rs](../../src/flashing/builder.rs),
[flasher.rs](../../src/flashing/flasher.rs), [flash_algorithm.rs](../../src/flashing/flash_algorithm.rs),
[encoder.rs](../../src/flashing/encoder.rs), [erase.rs](../../src/flashing/erase.rs),
[progress.rs](../../src/flashing/progress.rs), [error.rs](../../src/flashing/error.rs).

## 1. Concept

probe-rs programs non-volatile memory by running **CMSIS-Pack flash algorithms**
on the target itself: the algorithm blob is copied into RAM, the core's PC/argument
registers are set to call `Init`, `EraseSector`, `ProgramPage`, ... and the core
is run until it hits a breakpoint placed in the blob header. RAM regions are
written directly through the debug interface.

## 2. Components

```mermaid
classDiagram
    direction LR

    class ImageFormat {
        <<trait>>
        formats() names
        create_loader() ImageLoader
    }
    class ImageLoader {
        <<trait>>
        load(FlashLoader, Session, ImageReader)
    }
    class FlashLoader {
        -memory_map: Vec~MemoryRegion~
        -builder: FlashBuilder
        -source: TargetDescriptionSource
        -vector_table_addr: Option~u64~
        +add_data(addr, bytes)
        +load_image(session, reader, format, info)
        +commit(session, DownloadOptions)
        +verify(session, progress)
        +boot_info() BootInfo
        -prepare_plan() Vec~Flasher~
        -initialize()
        -verify_ram()
    }
    class FlashBuilder {
        data: BTreeMap addr to bytes
        build_sectors_and_pages()
    }
    class FlashLayout {
        sectors: Vec~FlashSector~
        pages: Vec~FlashPage~
        fills: Vec~FlashFill~
    }
    class Flasher {
        +core_index
        +flash_algorithm: FlashAlgorithm
        +regions: Vec~LoadedRegion~
        +init(session, clock) ActiveFlasher
        +run_erase_all()
        +run_erase() / run_program() / run_verify()
        -load(session)
        -program_simple() / program_double_buffer()
    }
    class ActiveFlasher~O~ {
        -core: Core
        +erase_all() / erase_sector()
        +program_page() / start_program_page_with_buffer()
        +wait_for_write_end()
        +call_vendor_function()
        -call_function_and_wait()
        -load_data()
        -check_for_stack_overflow()
        -read_rtt()
    }
    class FlashAlgorithm {
        load_address, instructions
        pc_init / pc_uninit
        pc_program_page / pc_erase_sector
        pc_erase_all / pc_verify / pc_blank_check / pc_read
        vendor_functions
        static_base, stack_top, stack_size
        page_buffers
        rtt_control_block
        +assemble_from_raw_with_data()
    }
    class Operation {
        <<trait>>
    }
    class Erase
    class Program
    class Verify

    ImageFormat ..> ImageLoader : creates
    ImageLoader ..> FlashLoader : fills
    FlashLoader *-- FlashBuilder
    FlashBuilder ..> FlashLayout
    FlashLoader ..> Flasher : prepare_plan
    Flasher *-- FlashAlgorithm
    Flasher ..> ActiveFlasher : init
    ActiveFlasher ..> Operation
    Erase ..|> Operation
    Program ..|> Operation
    Verify ..|> Operation
```

### Image formats

`static LOADERS` registry of `&'static dyn ImageFormat`:

| Format | Factory | Feature |
|---|---|---|
| ELF | `ElfLoaderFactory` | `builtin-formats` (uses `object`) |
| BIN (raw, base address + skip) | `BinLoaderFactory` | `builtin-formats` |
| Intel HEX | `HexLoaderFactory` | `builtin-formats` (uses `ihex`) |
| UF2 | `Uf2LoaderFactory` | `builtin-formats` (uses `uf2-decode`) |
| ESP-IDF | `IdfLoaderFactory` | registered by `probe-rs-espressif` plugin |

`flashing::image_format(name)` looks up a format by name;
`register_image_format` is used by the plugin system.

## 3. End-to-end flow

```mermaid
sequenceDiagram
    autonumber
    actor App
    participant D as download_file_with_options
    participant FL as FlashLoader
    participant FB as FlashBuilder
    participant F as Flasher (per algorithm)
    participant AF as ActiveFlasher
    participant C as Core
    participant T as Target RAM / NVM

    App->>D: (session, path, format, DownloadOptions)
    D->>FL: session.target().flash_loader()
    D->>FL: load_image(session, file, format)
    FL->>FB: add_data(addr, bytes) for each segment
    D->>FL: commit(session, options)

    FL->>FL: prepare_plan(): group NVM regions by algorithm<br/>(respect preferred_algos) → Vec of Flasher
    alt dry_run
        FL-->>App: Ok (nothing written)
    end
    FL->>F: initialize(): chip-erase support, layout,<br/>keep_unwritten_bytes, preverify

    loop each Flasher
        opt do_chip_erase (once per algorithm)
            F->>AF: run_erase_all → erase_all()
        end
        F->>F: program(session, progress, skip_erase, verify, double_buffering)
        F->>C: load(): reset_and_halt, write algorithm blob to load_address,<br/>fill stack with canary pattern
        F->>AF: init(clock) → call pc_init(addr, clock, op)
        loop each sector
            AF->>C: set PC = pc_erase_sector, r0 = addr, LR = BKPT header
            C->>T: run until halted
        end
        loop each page
            AF->>C: write page into page_buffer (load_data)
            AF->>C: call pc_program_page(addr, size, buffer)
            Note right of AF: double-buffering overlaps the next<br/>load_data with the running ProgramPage
        end
        opt verify
            F->>AF: run_verify (pc_verify or read-back compare)
        end
        AF->>C: pc_uninit, check_for_stack_overflow
    end

    FL->>C: if BootInfo::FromRam → reset_and_halt cores
    FL->>T: write RAM regions directly (halt core first)
    opt verify
        FL->>FL: verify_ram()
    end
    FL-->>App: Ok
```

## 4. Calling a flash algorithm routine

`ActiveFlasher::call_function` (in [flasher.rs](../../src/flashing/flasher.rs)):

```mermaid
flowchart LR
    A["Registers { pc, r0..r3 }"] --> B["write argument registers<br/>(argument_register(0..3))"]
    B --> C["write static base (core_register 9)<br/>and, on init, stack pointer = stack_top"]
    C --> D["return_address = load_address<br/>(+1 for Thumb)<br/>→ points at BKPT / EBREAK header"]
    D --> E["core.run()"]
    E --> F["wait_for_completion:<br/>poll status, read flasher RTT,<br/>until Halted"]
    F --> G["read return value (r0 / a0)<br/>error code (non-zero, or negative<br/>for some routines) → FlashError::RoutineCallFailed"]
```

The blob header is architecture specific: Thumb `BKPT` (`0xBE00BE00`), A32 `BKPT`,
`HLT` for ARMv8-A/R, two `EBREAK` for RISC-V, none for Xtensa.

Optional features of the algorithm:

* **Flasher RTT** — if `rtt_control_block` is set and enabled, algorithm log output
  is read during `wait_for_completion`.
* **Stack overflow check** — stack area pre-filled with a pattern and checked afterwards.
* **Vendor functions** — extra named entry points callable via `call_vendor_function`.
* **Double buffering** — when ≥ 2 `page_buffers` exist and not disabled.

## 5. `DownloadOptions`

| Field | Effect |
|---|---|
| `progress: FlashProgress` | Callback receiving `ProgressEvent`s (`FlashLayoutReady`, `AddProgressBar`, `Started`, `Progress`, `Finished`, `Failed`, `DiagnosticMessage`; operations `Fill`/`Erase`/`Program`/`Verify`) |
| `keep_unwritten_bytes` | Read back and re-write sector contents not covered by the image |
| `dry_run` | Plan only, write nothing |
| `do_chip_erase` | Erase whole chip (per algorithm) instead of sectors |
| `skip_erase` | Assume flash already erased |
| `preverify` | Compare before writing; skip if identical |
| `verify` | Verify after programming |
| `disable_double_buffering` | Force simple programming |
| `preferred_algos` | Choose among multiple algorithms covering a region |

## 6. Other entry points

* [erase.rs](../../src/flashing/erase.rs) — `erase_all`, `erase` (address range) and `run_blank_check`, without programming.
* `Session::sequence_erase_all()` — debug-sequence-based mass erase
  (`ArmDebugSequence::debug_erase_sequence`), used for unlocking.
* [encoder.rs](../../src/flashing/encoder.rs) — `FlashEncoder` / `TransferEncoding`
  (`Raw` or `Miniz`-compressed transfer for algorithms that support it).
