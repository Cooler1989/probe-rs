# 04 — Targets, vendors and plugins

Sources: [config/](../../src/config/mod.rs) ([target.rs](../../src/config/target.rs),
[registry.rs](../../src/config/registry.rs), [chip_info.rs](../../src/config/chip_info.rs)),
[vendor/](../../src/vendor/mod.rs), [plugin.rs](../../src/plugin.rs),
[build.rs](../../build.rs), [targets/](../../targets), and the `probe-rs-target` crate.

## 1. From YAML to `Target`

```mermaid
flowchart LR
    subgraph Build["Build time (build.rs, feature builtin-targets)"]
        Y["targets/*.yaml<br/>(~240 ChipFamily files)<br/>+ $PROBE_RS_TARGETS_DIR"] --> PT["probe_rs_target::process_targets"]
        PT --> BIN["OUT_DIR/targets.bincode"]
    end

    subgraph Runtime
        BIN -->|"include_bytes! + bincode decode"| BT["static BUILTIN_TARGETS<br/>RwLock of Vec ChipFamily"]
        PLG["Plugin.targets<br/>(add_builtin_target)"] --> BT
        BT --> REG["Registry::from_builtin_families()<br/>+ generic Cortex-M / RISC-V families"]
        USR["Registry::add_target_family_from_yaml<br/>(user --chip-description-path)"] --> REG
        REG -->|"get_target_by_name(name)"| T["Target::new(family, chip)"]
        REG -->|"get_target_by_chip_info(ChipInfo)"| T
    end

    T --> FA["resolve flash_algorithms<br/>(+ synthesize NVM regions for algo ranges)"]
    T --> DS["vendor::try_create_debug_sequence(chip)<br/>or Default*Sequence"]
    T --> RTT["rtt_scan_regions<br/>(ranges from YAML or all RAM)"]
```

### `Target` (resolved, runtime form)

| Field | Meaning |
|---|---|
| `name` | Chip name (e.g. `nRF52840_xxAA`) |
| `cores: Vec<probe_rs_target::Core>` | name, `CoreType`, `CoreAccessOptions` (ARM: AP, debug/CTI base, JTAG TAP; RISC-V: hart id, TAP; Xtensa: TAP) |
| `flash_algorithms: Vec<RawFlashAlgorithm>` | CMSIS flash algorithm blobs and metadata |
| `memory_map: Vec<MemoryRegion>` | `Ram`, `Nvm`, `Generic` regions (with core visibility, alias flag, access flags) |
| `debug_sequence: DebugSequence` | `Arm` / `Riscv` / `Xtensa` sequence `Arc` |
| `rtt_scan_regions: ScanRegion` | Where to search for an RTT control block |
| `jtag: Option<Jtag>` | Scan chain description, `force_scan_chain` flag |
| `default_format` | Default image format name (e.g. `idf` for ESP) |
| `source` | `BuiltIn`, `Generic` or `External` |

`Target::flash_loader()` creates an empty `FlashLoader` for this memory map (see
[05-flashing.md](./05-flashing.md)). Mixed-architecture targets are rejected
(`Target::architecture()` asserts all cores share an architecture).

### `TargetSelector`

```mermaid
flowchart LR
    A["&str / String"] -->|From| U["Unspecified(name)"]
    B["Target"] -->|From| S["Specified(target)"]
    C["() / None"] -->|From| AU["Auto"]
    U --> R["Registry lookup<br/>(exact, then unique prefix match;<br/>ChipNotFound / ChipNotUnique)"]
    S --> D["used directly"]
    AU --> AD["vendor::auto_determine_target"]
```

## 2. Vendors

```mermaid
classDiagram
    class Vendor {
        <<trait>>
        +try_create_debug_sequence(chip) Option~DebugSequence~
        +try_detect_arm_chip(registry, iface, ArmChipInfo) Option~String~
        +try_detect_riscv_chip(registry, iface, idcode) Option~String~
        +try_detect_xtensa_chip(registry, iface, idcode) Option~String~
        +try_detect_chip_from_probe(registry, probe) Option~String~
    }
    Vendor <|.. Amd
    Vendor <|.. Microchip
    Vendor <|.. Infineon
    Vendor <|.. Holtek
    Vendor <|.. SiliconLabs
    Vendor <|.. TexasInstruments
    Vendor <|.. NordicSemi
    Vendor <|.. Nuclei
    Vendor <|.. Nxp
    Vendor <|.. RaspberryPi
    Vendor <|.. St
    Vendor <|.. Vorago
    Vendor <|.. Sifive
    Vendor <|.. Sifli
    Vendor <|.. Renesas
    Vendor <|.. Wch
    Vendor <|.. Espressif : plugin crate
```

* Vendors are stored in `static VENDORS: LazyLock<RwLock<Vec<&'static dyn Vendor>>>`.
* `register_vendor` **prepends**, so plugin vendors override built-in ones.
* Each vendor module (e.g. [vendor/st/](../../src/vendor/st/mod.rs),
  [vendor/nordicsemi/](../../src/vendor/nordicsemi/mod.rs)) matches chips by name
  and returns specialised sequences, e.g. `stm32_armv7`, `stm32h7`, `stm32n6`,
  `nrf52`, `nrf53`, `nrf54l`, `nrf91`, `rp2040`/`rp235x`, NXP `nxp_armv6m`/`nxp_armv7m`/
  `nxp_armv8m`/`mcx`, Microchip `atsam`/`mec17xx`, Silabs `efm32xg1`/`efm32xg2`, etc.

### Typical things a vendor sequence does

| Need | Hook overridden |
|---|---|
| Multi-drop SWD / special DP power-up (RP2040) | `debug_port_setup`, `debug_port_start` |
| Unlock read-protected chip via mass erase (nRF APPROTECT, STM32 RDP, Atmel) | `debug_device_unlock`, `debug_erase_sequence` (requires `Permissions::allow_erase_all`) |
| Keep debug alive in low power (STM32 DBGMCU) | `debug_core_start`, `debug_core_stop` |
| Non-standard reset (AIRCR not enough, watchdog, boot ROM) | `reset_system`, `reset_catch_set` |
| Trace clock / pin muxing | `trace_start` |
| Remap memory before running from RAM | `prepare_running_on_ram` |
| Hide inaccessible APs | `allowed_access_ports` |

## 3. Chip auto-detection (`TargetSelector::Auto`)

```mermaid
sequenceDiagram
    autonumber
    participant S as Session::new
    participant P as Probe
    participant V as vendor::auto_determine_target
    participant VS as Vendors (in order)
    participant R as Registry

    S->>P: (UnderReset? target_reset_assert)
    S->>P: attach_to_unspecified()
    S->>V: auto_determine_target(registry, probe)

    Note over V: 1. ARM
    V->>P: try_into_arm_debug_interface(DefaultArmSequence)
    V->>V: select_debug_port(Default) + read_chip_info_from_rom_table
    loop each vendor
        V->>VS: try_detect_arm_chip(registry, iface, ArmChipInfo)
    end
    alt no vendor match
        V->>R: get_target_by_chip_info(JEP106 + part)
    end
    V->>P: interface.close() → Probe

    Note over V: 2. RISC-V (if none yet)
    V->>P: riscv builder.attach + enter_debug_mode + read_idcode
    V->>VS: try_detect_riscv_chip(...)

    Note over V: 3. Xtensa (if none yet)
    V->>P: xtensa iface enter_debug_mode + read_idcode
    V->>VS: try_detect_xtensa_chip(...)

    Note over V: 4. Probe firmware metadata
    V->>VS: try_detect_chip_from_probe(...)

    V->>P: detach()
    V-->>S: (Probe, Option Target)
    S->>P: (UnderReset? target_reset_deassert)
```

Failure yields `Error::ChipNotFound(RegistryError::ChipAutodetectFailed)`.

## 4. Plugin system

```mermaid
flowchart LR
    PL["Plugin {<br/>vendors, image_formats,<br/>targets, probe_drivers }"] -->|"register_plugin(plugin)"| RP["plugin.rs"]
    RP --> V["vendor::register_vendor<br/>(prepend)"]
    RP --> IF["flashing::register_image_format"]
    RP --> TG["config::registry::add_builtin_target"]
    RP --> PF["probe::register_probe_factory<br/>(append)"]
```

Plugin crates in this workspace:

* **probe-rs-espressif** — registers vendor `Espressif` (chip detection, Xtensa and
  RISC-V sequences), probe driver `EspUsbJtagFactory`, image format `IdfLoaderFactory`
  (ESP-IDF), and its own bincode-embedded targets.
* **probe-rs-linux** — Linux GPIO (`linuxgpiod`) and spidev SWD bit-bang probe drivers.

Because all extension points are global registries, a plugin must be registered
**before** the first `Lister`/`Registry`/`Session` use that should see it.
