# 02 — Probe layer

Sources: [probe.rs](../../src/probe.rs), [probe/list.rs](../../src/probe/list.rs),
[probe/selector.rs](../../src/probe/selector.rs), [probe/common.rs](../../src/probe/common.rs),
[probe/queue.rs](../../src/probe/queue.rs), [probe/usb_util.rs](../../src/probe/usb_util.rs),
driver folders under [probe/](../../src/probe/).

## 1. Responsibilities

* Enumerate connected debug probes (USB via `nusb`/`hidapi`, serial ports, network for Glasgow/BMP).
* Open a probe from a `DebugProbeSelector` (`VID:PID[:serial]`, interface).
* Configure wire protocol (`WireProtocol::Swd` / `Jtag`), speed and reset lines.
* Expose **capabilities** to the architecture layer: an ARM debug interface, a
  RISC-V DTM builder, an Xtensa interface, a JTAG chain, SWO.

## 2. Discovery and opening

```mermaid
flowchart LR
    subgraph Registry["static DRIVERS: global list of dyn ProbeFactory"]
        BMP["BlackMagicProbeFactory"]
        CMSIS["CmsisDapFactory"]
        FTDI["FtdiProbeFactory"]
        STL["StLinkFactory"]
        JL["JLinkFactory"]
        WCH["WchLinkFactory"]
        SIF["SifliUartFactory"]
        GL["GlasgowFactory"]
        CH["Ch347UsbJtagFactory"]
        PLUG["... plugin factories<br/>(register_probe_factory)"]
    end

    Lister["Lister<br/>(Box dyn ProbeLister)"] --> APL["AllProbesLister"]
    APL -->|"list_probes()"| Registry
    Registry -->|"Vec of DebugProbeInfo"| Info["DebugProbeInfo<br/>identifier, vid, pid, serial,<br/>interface, probe_factory"]
    Info -->|"open()"| Fac["probe_factory.open(selector)"]
    Fac --> DP["Box dyn DebugProbe"] --> Probe["Probe { inner, attached }"]
```

* `Lister::new()` uses `AllProbesLister`; `Lister::with_lister()` lets tools inject a
  custom/limited lister (e.g. the RPC server's `LimitedLister`, test mocks).
* `register_probe_factory` (crate-private, used by [plugin.rs](../../src/plugin.rs))
  appends extra drivers such as `probe-rs-espressif`'s ESP USB-JTAG and
  `probe-rs-linux`'s GPIO / spidev bit-bangers.

## 3. Trait hierarchy

```mermaid
classDiagram
    direction BT

    class ProbeFactory {
        <<trait>>
        +open(selector) Box dyn DebugProbe
        +list_probes() Vec~DebugProbeInfo~
        +list_probes_filtered(selector)
    }

    class DebugProbe {
        <<trait>>
        +get_name()
        +speed_khz() / set_speed()
        +attach() / detach()
        +target_reset() / _assert() / _deassert()
        +select_protocol() / active_protocol()
        +try_get_arm_debug_interface(seq)
        +try_get_riscv_interface_builder()
        +try_get_xtensa_interface(state)
        +try_as_jtag_probe()
        +try_as_dap_probe()
        +get_swo_interface()
        +get_target_voltage()
        +into_probe()
    }

    class RawSwdIo {
        <<trait>>
        +swd_io(seq) Vec~bool~
        +swj_pins()
        +swd_settings()
    }
    class RawJtagIo {
        <<trait>>
        +state_mut() JtagDriverState
        +shift_bit() / shift_bits() / read_captured_bits()
    }
    class AutoImplementJtagAccess {
        <<marker trait>>
    }
    class JtagAccess {
        <<trait>>
        +set_scan_chain / set_expected_scan_chain
        +scan_chain() / select_target(idx)
        +tap_reset() / shift_raw_sequence()
        +read_register / write_register / write_dr
        +write_register_batch(ErasedQueue)
    }
    class RawDapAccess {
        <<trait (arm::traits)>>
        +raw_read_register / raw_write_register
        +raw_read_block / raw_write_block
        +swj_sequence / swj_pins / jtag_sequence
    }
    class DapProbe {
        <<trait>>
    }
    class SwoAccess {
        <<trait (arm::swo)>>
        +enable_swo / disable_swo / read_swo
    }

    RawSwdIo --|> DebugProbe
    RawJtagIo --|> DebugProbe
    JtagAccess --|> DebugProbe
    AutoImplementJtagAccess --|> RawJtagIo
    DapProbe --|> DebugProbe
    DapProbe --|> RawDapAccess

    AutoImplementJtagAccess ..> JtagAccess : blanket impl (probe/common.rs)
    RawSwdIo ..> RawDapAccess : blanket impl with JtagAccess (arm/traits/polyfill.rs)
```

### Blanket implementations ("polyfills")

| Provided | Requirement | Location |
|---|---|---|
| `JtagAccess` | `T: AutoImplementJtagAccess` (i.e. `RawJtagIo`) | [probe/common.rs](../../src/probe/common.rs) — implements TAP state machine, IR/DR shifting, scan-chain detection on top of bit-level IO |
| `RawDapAccess` | `T: DebugProbe + RawSwdIo + JtagAccess` | [architecture/arm/traits/polyfill.rs](../../src/architecture/arm/traits/polyfill.rs) — implements ADI SWD/JTAG-DP transfers (ACK/WAIT/retry, parity) from raw bits |

This lets a bit-level driver (FTDI, J-Link, BMP, CH347, Linux GPIO) gain full ARM
DAP support by implementing only `RawSwdIo` + `RawJtagIo`.

## 4. Driver capability matrix

Derived from `impl ... for ...` blocks in each driver folder:

| Driver | `DebugProbe` | `RawSwdIo` | `RawJtagIo` → `JtagAccess` | native `JtagAccess` | `RawDapAccess` | own `ArmDebugInterface` | `SwoAccess` |
|---|---|---|---|---|---|---|---|
| CMSIS-DAP (v1 HID / v2 bulk) | ✔ | | ✔ | | ✔ native DAP commands | | ✔ |
| SEGGER J-Link | ✔ | ✔ | ✔ | | polyfill | | ✔ |
| FTDI (MPSSE) | ✔ | ✔ | ✔ | | polyfill | | |
| Black Magic Probe | ✔ | ✔ | ✔ | | polyfill | ✔ (remote protocol) | ✔ |
| CH347 USB-JTAG | ✔ | ✔ | ✔ | | polyfill | | |
| Glasgow | ✔ | | | | ✔ native | | |
| ST-Link | ✔ | | | | | ✔ `StlinkArmDebug` (high-level API) | ✔ |
| WCH-Link | ✔ | | | ✔ | | | |
| SiFli UART | ✔ | | | | | ✔ `SifliUartArmDebug` | ✔ |
| `FakeProbe` (tests) | ✔ | | | | | ✔ `FakeArmInterface` | |

Two strategies exist for ARM:

1. **Generic ADI stack** — the probe supplies `RawDapAccess` (natively or via
   polyfill) and `try_get_arm_debug_interface` returns an
   `ArmCommunicationInterface` which implements DP/AP logic in probe-rs.
2. **Probe-managed ADI** — probes with high-level firmware (ST-Link, BMP remote,
   SiFli) return their own `ArmDebugInterface` implementation.

## 5. Probe lifecycle

```mermaid
stateDiagram-v2
    [*] --> Opened : factory.open()
    Opened --> Configured : select_protocol / set_speed
    Configured --> Attached : attach_to_unspecified()
    Opened --> Attached : attach_to_unspecified()
    Attached --> ArmIface : try_into_arm_debug_interface(seq)
    Attached --> JtagSession : kept inside ArchitectureInterface.Jtag
    ArmIface --> Attached : ArmDebugInterface.close() returns Probe
    Attached --> SessionOwned : Session.new(...)
    SessionOwned --> [*] : drop(Session)
```

`Probe` public API (selection): `attach`, `attach_with_registry`,
`attach_under_reset[_with_registry]`, `attach_to_unspecified[_under_reset]`,
`select_protocol`, `set_speed`, `target_reset[_assert|_deassert]`, `detach`,
`try_into_arm_debug_interface`, `try_get_riscv_interface_builder`,
`try_get_xtensa_interface`, `try_as_jtag_probe`, `try_as_dap_probe`,
`get_swo_interface[_mut]`, `get_target_voltage`.

`AttachMethod` is `Normal` (default) or `UnderReset` (hold nRESET while connecting —
needed when firmware disables debug pins or sleeps).

## 6. JTAG command batching

[probe/queue.rs](../../src/probe/queue.rs) defines a typed `Queue<E>` of
`JtagCommand`s (register writes / DR shifts with result transformers) and its
type-erased form `ErasedQueue`. Drivers execute it via
`JtagAccess::write_register_batch(&ErasedQueue)`, returning a
`DeferredResultSet<CommandResult>` indexed by `DeferredResultIndex`; partial
failures are reported as `BatchExecutionError` carrying the results gathered so
far. The default implementation executes commands one by one (and logs a
performance warning). The RISC-V JTAG DTM relies heavily on batching to
pipeline debug module accesses.

## 7. Errors

```mermaid
flowchart LR
    PE["dyn ProbeError<br/>(driver-specific, e.g. StlinkError)"] --> BPE["BoxedProbeError"]
    BPE --> DPE["DebugProbeError<br/>(Usb, ProbeSpecific, ProbeCouldNotBeCreated,<br/>UnsupportedProtocol, InterfaceNotAvailable,<br/>NotAttached, Timeout, JtagScanChain, Other ...)"]
    PCE["ProbeCreationError<br/>(NotFound, CouldNotOpen, HidApi, Usb, ...)"] --> DPE
    DPE --> ERR["crate::Error::Probe"]
```
