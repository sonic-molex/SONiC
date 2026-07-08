# `xcvrd` Developer Guide (sonic-platform-daemons)

Source: `~/ocs/sonic-molex/sonic-buildimage/src/sonic-platform-daemons/sonic-xcvrd`

Internals of the SONiC transceiver daemon — threading, state machines, the CMIS datapath, the DB contract, and how to add a new transceiver type. For the operator-facing CLI/telemetry surface (no CMIS knowledge required), see the companion **[Operator Guide](./xcvrd-operator-guide.md)**.

## Purpose

`xcvrd` owns everything about pluggable optics (SFP/QSFP/QSFP-DD/OSFP): detecting insert/remove, reading EEPROM/DOM/VDM data, publishing it to Redis, applying media/SI settings, and driving CMIS modules through their datapath bring-up. It's a single Python process (`docker-pmon`) that runs a main daemon plus several worker threads. It is a **producer** into `STATE_DB`; all CLI/northbound interfaces are separate components that read those tables.

**Important:** `xcvrd` does **not** talk to module I2C (or cage GPIO) directly. All hardware access goes through the **platform API** (`platform_chassis.get_sfp()` → `Sfp.read_eeprom()`, presence, change events, CMIS helpers). Your platform package (or legacy `sfputil`) owns the bus — host I2C sysfs, switch-ASIC module management, BMC, or **FPGA mailbox** over PCIe.

## DOM, VDM, and status (developer view)

Operator-facing definitions of **DOM**, **VDM**, and **status** are in the **[Operator Guide](./xcvrd-operator-guide.md#background-dom-vdm-and-status-monitoring)**. From a developer perspective:

| Concern | Primary `xcvrd` owner | DB / helpers |
|--------|------------------------|--------------|
| **DOM** (temperature, voltage, TX/RX power, bias) | `DomInfoUpdateTask` | `TRANSCEIVER_DOM_*` tables; `dom/utilities/dom_sensor/` |
| **VDM** (Pre-FEC BER, SNR, coherent metrics, flags) | `DomInfoUpdateTask` (+ threshold boot path in `SfpStateUpdateTask`) | `TRANSCEIVER_VDM_*`; `dom/utilities/vdm/` |
| **Status** (module state, alarms, presence) | `SfpStateUpdateTask`, `CmisManagerTask`, `DomInfoUpdateTask` | `TRANSCEIVER_STATUS`, `TRANSCEIVER_STATUS_SW` (incl. `cmis_state`) |

Flow: platform **`Sfp`** APIs → parse via **`sonic_platform_base.sonic_xcvr`** → publish to **`STATE_DB`** → CLI / telemetry / orchagent consumers.

## Platform layer and optics hardware access

### Where the bus lives

```text
xcvrd  →  sonic_platform (Sfp API)  →  [your transport]  →  module EEPROM / GPIO
```

Stock SONiC platforms use different transports; none of them require `xcvrd` to know which one you use:

| Transport | Example |
|-----------|---------|
| Linux **optoe** + I2C mux | `/sys/bus/i2c/devices/.../eeprom` via `SfpOptoeBase` |
| **Switch ASIC** module management | Mellanox-style MCIA over PCI (not host cage I2C) |
| **BMC / OOB** | IPMI or custom daemon; platform wraps it |
| **FPGA mailbox** | PCIe BAR mmap → SPI/I2C on another board (see below) |

If the **control CPU has no I2C to the cages**, implement **`read_eeprom()` / `get_presence()` / `get_change_event()`** in **`sonic_platform`** over whatever path *does* reach the modules. Do **not** patch `xcvrd` to open `/dev/i2c-*` on the CPU.

### Molex reference topology (control CPU + single line card)

Many OTN/Molex designs separate the control CPU from cage I2C:

```text
CPU (SONiC) ──PCIe (BAR in CPU memory)──► Control-card FPGA ──SPI (backplane)──► Line-card FPGA ──I2C/GPIO──► Transceivers
```

With **one line card**, software does not need slot routing — only **cage / port index** (0…N−1). The control FPGA can hard-code **line card 0** on SPI.

| Location | Responsibility |
|----------|----------------|
| **Line-card FPGA** | I2C master to module EEPROM (`0x50` / `0x51`), **ModSel**, **LPMode**, **Reset**, **presence**, optional insert/remove debounce |
| **Control-card FPGA** | Forward **port-indexed** read/write/presence over SPI; timeouts; optional presence cache |
| **CPU Linux** | Map PCIe BAR; expose mailbox to user space (driver or direct mmap of `resource0`) |
| **`sonic_platform`** | Map logical port → cage index; implement `SfpBase` / `SfpOptoeBase` API |
| **`xcvrd`** | Unchanged — runs on the control CPU if the platform API is complete |

### Control FPGA already mapped in CPU memory

If the control FPGA is **already mmap'd** into the CPU address space (PCIe BAR via a kernel driver or `/sys/bus/pci/devices/<BDF>/resource0`), treat the BAR as a **register mailbox** and keep all SPI/I2C details inside FPGA firmware.

**1. Centralize BAR access** — one small library (e.g. `fpga_xcvr.py` or vendor `fpga_lib`) used only from `sonic_platform`:

- Open/mmap **`resource0`** once at init.
- **`read32` / `write32`** at fixed offsets (use **volatile** access; match FPGA endianness, usually little-endian on x86/ARM).
- **Process-wide lock** — `xcvrd` uses multiple threads; serialize mailbox transactions so CLI/debug tools cannot interleave SPI frames.

Reference pattern in-tree: Nexthop **`platform/broadcom/sonic-platform-modules-nexthop/common/nexthop/fpga_lib.py`** (PCIe `resource0` register read/write).

**2. Suggested mailbox layout** (adjust to your RTL):

| Offset | Register | Purpose |
|--------|----------|---------|
| `0x00` | `CMD` | IDLE / READ / WRITE / GET_PRESENCE / SET_LPMODE / … |
| `0x04` | `PORT` | Cage index on the line card |
| `0x08` | `I2C_ADDR` | `0x50` or `0x51` |
| `0x0C` | `OFFSET` | EEPROM byte offset |
| `0x10` | `LEN` | Transfer length |
| `0x14` | `STATUS` | OK / busy / error / timeout |
| `0x18+` | `DATA[]` | Payload |

CPU: fill `PORT`…`LEN` → write `CMD` → poll `STATUS` (or MSI) → read `DATA`. Control FPGA completes the transaction over SPI to the line FPGA.

**3. CMIS paging** — for offset ≥ 128, line FPGA or driver must **select the CMIS page** (byte 127 on `0x50`) before read/write, same as the Linux **optoe** driver would on a direct-I2C design.

**4. `sonic_platform` minimum for `xcvrd`**

- `Chassis.get_num_sfps()` / `get_sfp(physical_port)`
- `read_eeprom()` / `write_eeprom()` via mailbox
- `get_presence()` — GPIO bitmap or `GET_PRESENCE`
- `get_change_event(timeout)` — poll presence delta or FPGA event queue
- CMIS sideband as needed: **LP mode**, reset, etc. (for `CmisManagerTask`)

**5. Config** — `platform.json` + `port_config.ini`: logical Ethernet port *i* → cage *i* (single line card).

### Bring-up order (FPGA path)

1. Line FPGA: read CMIS/SFF **page 0** on one cage over I2C.
2. SPI: control ↔ line **READ** for port 0.
3. CPU: userspace tool via mmap reads the same bytes.
4. `sonic_platform`: one port, then all ports.
5. Start **`xcvrd`**; confirm **`STATE_DB`** `TRANSCEIVER_INFO`.
6. Enable **`CmisManagerTask`** / DOM once EEPROM and presence are stable.

### Pitfalls

- **Locking** — one mutex for all BAR/SPI access from `xcvrd` and tools.
- **Timeouts** — hung SPI must return an error to platform code, not block `DomInfoUpdateTask` indefinitely.
- **Writes** — CMIS bring-up and tuning need **`write_eeprom()`** early, not read-only bring-up.
- **Hot-plug** — debounced presence on the line FPGA; expose via **`get_change_event()`**.
- **Production** — prefer a **PCIe kernel driver** mmap over raw `/dev/mem` when possible.

## Process & threading architecture

The entry point is `DaemonXcvrd.run()` (`xcvrd.py:1141`). After init it spawns up to **five child threads**, then just waits on `stop_event` (`xcvrd.py:1186`):

```python
# Start the SFF manager
sff_manager = None
if self.enable_sff_mgr:
    sff_manager = SffManagerTask(...)
...
if not self.skip_cmis_mgr:
    cmis_manager = CmisManagerTask(...)
...
dom_info_update = DomInfoUpdateTask(...)
...
if self.dom_temperature_poll_interval is not None:
    dom_thermal_info_update = DomThermalInfoUpdateTask(...)
...
sfp_state_update = SfpStateUpdateTask(...)
```

| Thread | Source | Job |
|---|---|---|
| `SfpStateUpdateTask` | `xcvrd.py:259` | Core insert/remove event loop; populates `TRANSCEIVER_INFO` + DOM/VDM thresholds on plug events |
| `CmisManagerTask` | `cmis/cmis_manager_task.py:41` | Per-port CMIS state machine that configures/activates the datapath of CMIS optics |
| `DomInfoUpdateTask` | `dom/dom_mgr.py` | Periodically polls live DOM sensors, VDM, and status flags into the DB |
| `DomThermalInfoUpdateTask` | `dom/dom_mgr.py` | Optional, faster temperature-only polling (only if `--dom_temperature_poll_interval` given) |
| `SffManagerTask` | `sff_mgr.py:45` | Optional (`--enable_sff_mgr`); deterministic link bring-up for non-CMIS SFF modules via `host_tx_ready`/Tx-disable |

All threads talk to the hardware through the **platform API** (`platform_chassis.get_sfp(physical_port)` → an `Sfp` object with `get_transceiver_info()`, CMIS/SFF APIs, etc.), falling back to the legacy `platform_sfputil` plugin if no chassis class exists (`xcvrd.py:114`, `xcvrd.py:1027`).

## Init sequence (`DaemonXcvrd.init`, `xcvrd.py:1020`)

1. Load `sonic_platform.platform.Platform().get_chassis()` (or legacy `sfputil`).
2. Resolve front-end namespaces (multi-ASIC aware) and build the `XcvrTableHelper` (one set of Redis table handles per ASIC).
3. Unless fast-reboot, load `media_settings.json` and `optics_si_settings.json`.
4. **Wait for port config done** — blocks on `APPL_DB PORT_TABLE` until `PortConfigDone`/`PortInitDone` (`xcvrd.py:915`) so it doesn't run before ports exist.
5. Build the port mapping, seed `STATE_DB PORT_TABLE` with `NPU_SI_SETTINGS_SYNC_STATUS`, build `sfp_obj_dict`, and remove stale `TRANSCEIVER_INFO` for absent modules.

## `SfpStateUpdateTask` — the heart of it

This is the classic xcvrd event loop. `task_worker` (`xcvrd.py:394`) runs a small **state machine** (`INIT → NORMAL → EXIT`) driven by events from `_wrapper_get_transceiver_change_event()`:

- States: `STATE_INIT`, `STATE_NORMAL`, `STATE_EXIT` (`xcvrd.py:73`).
- Events: `SYSTEM_NOT_READY`, `SYSTEM_BECOME_READY`, `SYSTEM_FAIL`, `NORMAL_EVENT` (`xcvrd.py:68`), with the full transition table documented inline at `xcvrd.py:455`.
- It retries up to `RETRY_TIMES_FOR_SYSTEM_READY` (24 × 5s) waiting for the platform to become ready; on repeated failure it `SIGTERM`s the parent and exits.

On a **`NORMAL_EVENT`** (`xcvrd.py:522`) it maps physical→logical ports and, per port:

- **Inserted**: marks `TRANSCEIVER_STATUS_SW` inserted, calls `post_port_sfp_info_to_db()` (EEPROM → `TRANSCEIVER_INFO`), posts DOM + VDM thresholds, and applies media settings. If the EEPROM isn't ready it retries once, then adds the port to `retry_eeprom_set` for slow background retry (`retry_eeprom_reading`, `xcvrd.py:836`).
- **Removed**: drops the xcvr API object and deletes that port's rows from ~25 DOM/VDM/status/PM tables (`xcvrd.py:573`).
- **Error bits**: decodes the error, writes a description into `TRANSCEIVER_STATUS_SW`, and clears DOM info if the error blocks EEPROM reads.

It also subscribes to `CONFIG_DB` port changes (`on_port_config_change`, `xcvrd.py:722`) to add/remove logical ports dynamically (breakout, etc.). Because it can sleep in platform calls, shutdown uses an async exception injection (`raise_exception`, `xcvrd.py:709`) for graceful join.

`post_port_sfp_info_to_db` (`xcvrd.py:178`) is the shared helper that branches on whether the module is CMIS (`'cmis_rev' in port_info_dict`) vs. legacy SFF when building the `FieldValuePairs` for `TRANSCEIVER_INFO`.

## `CmisManagerTask` — CMIS datapath state machine

For CMIS modules (`QSFP-DD/OSFP/...`, `cmis_manager_task.py:45`) this thread runs a **per-port state machine** through the CMIS host/media datapath bring-up:

```python
CMIS_STATE_UNKNOWN, CMIS_STATE_INSERTED, CMIS_STATE_DP_PRE_INIT_CHECK,
CMIS_STATE_DP_DEINIT, CMIS_STATE_AP_CONF, CMIS_STATE_DP_ACTIVATE,
CMIS_STATE_DP_INIT, CMIS_STATE_DP_TXON, CMIS_STATE_READY,
CMIS_STATE_REMOVED, CMIS_STATE_FAILED, CMIS_TERMINAL_STATES
```

It watches `CONFIG_DB PORT` + `STATE_DB PORT_TABLE` (e.g. `host_tx_ready`) via `PortChangeObserver`, and for each port selects the correct application/datapath, deinitializes lanes, applies app/SI config, activates and turns Tx on, then lands in `CMIS_STATE_READY`. It records progress in `TRANSCEIVER_STATUS_SW`'s `cmis_state` field (`update_port_transceiver_status_table_sw_cmis_state`, `cmis_manager_task.py:85`), retries up to `CMIS_MAX_RETRIES`, and uses expiration timers (`CMIS_DEF_EXPIRED`). It's gear-box aware and honors fast-reboot.

## DOM / thermal polling

`DomInfoUpdateTask` (`dom/dom_mgr.py:37`) periodically refreshes live diagnostics into the DB — DOM sensors, VDM real values/flags, and status flags — using the helper utility classes (`DOMDBUtils`, `VDMDBUtils`, `StatusDBUtils`). It honors a per-port `dom_polling = disabled` knob from `CONFIG_DB PORT` (`get_dom_polling_from_config_db`, `dom/dom_mgr.py:74`) and reacts to port removal. `DomThermalInfoUpdateTask` is a lighter temperature-only variant for faster thermal updates.

Default interval is **60 s** (`DEFAULT_DOM_INFO_UPDATE_PERIOD_SECS`, overridable via `--dom_update_interval`). Each cycle costs ~1,530 bytes across ~192 individual EEPROM reads per port, **~93% of it VDM** — measured breakdown in [Monitoring data footprint](#monitoring-data-footprint).

## DB footprint

Everything is keyed by logical port and routed per-ASIC through `XcvrTableHelper`. Reads come from `CONFIG_DB`/`APPL_DB` (port config, port-config-done); writes go to **STATE_DB**: `TRANSCEIVER_INFO`, `TRANSCEIVER_DOM_SENSOR`/threshold/flag tables, `TRANSCEIVER_VDM_*`, `TRANSCEIVER_STATUS`/`TRANSCEIVER_STATUS_SW`, `TRANSCEIVER_PM`, `TRANSCEIVER_FIRMWARE_INFO`, and `PORT_TABLE` (`NPU_SI_SETTINGS_SYNC_STATUS`, `host_tx_ready`).

## Shutdown

On `SIGINT`/`SIGTERM` the handler sets `stop_event` (`xcvrd.py:901`); the main loop joins all threads (injecting the async exception into `SfpStateUpdateTask`), then `deinit()` (`xcvrd.py:1081`) clears most transceiver tables — but deliberately **skips deleting `TRANSCEIVER_INFO`** and, on warm/fast reboot, keeps `TRANSCEIVER_STATUS` so orchagent doesn't see a spurious Tx-disable.

## CPO / MC modules over CMIS

A **Media Channel (MC)** in the Molex CPO (co-packaged optics) design is a single co-packaged module that contains **both line-side and client-side optics**. Unlike a pluggable that has an electrical *host* side and an optical *media* side, an MC is optical on **both** sides, so it stretches the standard CMIS host↔media abstraction. This section captures how a CPO MC maps onto the CMIS model that `xcvrd` / `CmisManagerTask` already understands, and how big the monitoring footprint is.

### What CMIS defines (and doesn't)

CMIS (Common Management Interface Specification, "SEE-miss") defines the **logical** management interface: the paged 128-byte memory model, the module/data-path state machines, CDB messaging, and register semantics on a **2-wire (I2C-like) interface at address `1010000b` / `0xA0`**. It does **not** define the physical/electrical bus (I2C timing, pull-ups, clock speed) or the hardware control pins (`ResetL`, `LPMode`, `ModSelL`, `IntL`, `ModPrsL`) — those live in the **form-factor MSA** (QSFP-DD, OSFP, OSFP-1600, …). So: **CMIS = firmware/register contract; form-factor MSA = physical/electrical contract.** `xcvrd` only ever sees the CMIS side, via the platform API.

### Example MC: 1.6T line + 1.6T client (16 lanes)

For a Molex MC built from two **OSFP-1600-class** engines — **1.6T = 8 × 200G** per side:

| Side | Rate | Lanes |
|------|------|-------|
| Line | 1.6T (8×200G PAM4) | 8 |
| Client | 1.6T (8×200G PAM4) | 8 |
| **Total** | **3.2T** | **16** |

### Mapping to CMIS over a single I2C

With **one I2C interface to the MC you get exactly one CMIS memory space** (one `0xA0` device). All 16 lanes must be projected into that single paged memory — they do **not** get 16 addresses. CMIS expresses >8 lanes through **banking** (bank-select byte `126`), where each bank covers 8 lanes:

| Bank | Lanes | CPO mapping (example) |
|------|-------|-------------|
| **Bank 0** | 0–7 | **Client side** (1.6T, 8×200G) |
| **Bank 1** | 8–15 | **Line side** (1.6T, 8×200G) |

> **The bank↔side assignment is a documented vendor convention, not fixed by CMIS.** CMIS banks are just lane groups of 8 (Bank 0 = lanes 0–7, Bank 1 = lanes 8–15); which physical side lands on which bank is the module vendor's choice and **must be published in a vendor page / app descriptors** so platform code reads the right lanes. The client/line split above is only an example — pick one and make it authoritative.

A single `0xA0` memory space supports up to **4 banks = 32 lanes**, so a 16-lane MC fits cleanly in **2 banks**. The bank-aware pages repeat per bank:

| Page | Content | Banked? |
|------|---------|---------|
| 00h (lower) | Module temp/Vcc/aux, page + bank select | No (global) |
| 10h | Per-lane control (Tx enable, ApSel, …) | Yes (per bank) |
| 11h | Per-lane monitors + flags (Tx pwr/bias, Rx pwr, LOS/LOL) | Yes (per bank) |

To read all monitors: set **bank-select** → set **page-select (byte 127)** → read the 128-byte page. That is the *bus mechanism*; it is **not** how `xcvrd` actually reads. `sonic_xcvr` never reads a whole page for DOM — it issues one small read per field (1/2/4/16 bytes), so a single bank costs ~190 transactions per cycle, not one page-read. See [Monitoring data footprint](#monitoring-data-footprint).

CMIS has no first-class "line vs client" field for this CPO case, so carry the distinction by **bank grouping + Application codes / ApSel** (Page 00h app descriptors + Page 10h ApSel), and **document the lane/bank convention in a vendor page**.

### Monitoring data footprint

> **These are measured numbers, not estimates.** Obtained by wrapping `XcvrEeprom`'s reader with a recorder and replaying the exact call sequence `DomInfoUpdateTask` performs per port (`dom/dom_mgr.py:345-417`) against `sonic-otn/sonic-buildimage/src/sonic-platform-common`. Model: 8-lane CMIS 5.0 module advertising all per-lane monitors and flags, plus VDM with **2 descriptor pages** (16 observable types × 8 lanes). Re-measure if your MC's VDM advertisement differs — VDM dominates the total.

**Per 8-lane bank, per DOM cycle:**

| Poll step (STATE_DB table) | Reads | Bytes |
|---|---|---|
| `get_transceiver_dom_real_value` (`TRANSCEIVER_DOM_SENSOR`) | 15 | 67 |
| `get_transceiver_dom_flags` (`TRANSCEIVER_DOM_FLAG`) | 7 | 16 |
| `get_transceiver_status` (`TRANSCEIVER_STATUS`) | 11 | 17 |
| `get_transceiver_status_flags` (`TRANSCEIVER_STATUS_FLAG`) | 13 | 13 |
| VDM freeze + statistic-support probe | 5 | 259 |
| VDM statistic real values | 52 | 354 |
| VDM basic real values | 84 | 418 |
| VDM flags (`TRANSCEIVER_VDM_*_FLAG`) | 5 | 386 |
| **Total** | **192** | **1,530** |

By CMIS page: `20h`/`21h` (VDM descriptors) **512 bytes each**, `24h`/`25h` (VDM values) 128 each, `2Ch` (VDM flags) 128, and 122 bytes total across lower `00h`, `01h`, `02h`, `10h`, `11h`, `2Fh`.

**DOM/status is only 113 bytes of that — VDM is 93%.** Register-level estimates of this section's footprint tend to count only the DOM/status registers and land near ~156 bytes for 16 lanes; that is ~20× low because it omits VDM, which `DomInfoUpdateTask` polls every cycle. The per-register intuition itself holds up well against measurement (~12 bytes of module monitors, **6 bytes/lane** analog issued as three 16-byte block reads, ~**5.75 bytes/lane** of flags) — it just isn't the whole poll.

**Sensitivity — VDM advertisement drives everything:**

| Module VDM advertisement | Reads/bank | Bytes/bank |
|---|---|---|
| 2 descriptor pages | 192 | 1,530 |
| 1 descriptor page | 124 | 890 |
| VDM unsupported | 46 | 113 |

**Scaled to a 16-lane MC and a 17-MC shelf** (2 banks/MC — see the bank caveat under [How an MC maps to SONiC ports](#how-an-mc-maps-to-sonic-ports)):

| Quantity | Value |
|----------|-------|
| Per bank | 1,530 bytes / 192 transactions |
| **Per MC (2 banks)** | **~3.0 KB / ~384 transactions** |
| **17 MCs total** | **~52 KB / ~6,500 transactions** |
| Total lanes | 17 × 16 = **272** |
| 17 MCs, 1 VDM descriptor page | ~30 KB |
| 17 MCs, VDM unsupported | ~3.8 KB |

**Size the poll interval on transactions, not bytes.** 52 KB per 60 s cycle is nothing; ~6,500 individual reads is not. Each one needs a page-select write, plus mux-channel and bank selects on the [muxed topology](#scaling-beyond-one-cmis-space), and on the [FPGA-mailbox path](#platform-layer-and-optics-hardware-access) each is a PCIe round trip to a SPI→I2C bridge. That number, not the byte count, is what has to fit in the interval.

**Two independent optimizations — one saves bytes, the other saves transactions.** Fix both in `sonic-platform-common` (align with `sonic-otn` first), not as a platform-local workaround.

1. **Cache the VDM descriptor pages.** Every VDM operation re-reads the full 128-byte descriptor page before doing anything (`get_vdm_page`, `api/public/cmisVDM.py:72`; also `is_vdm_statistic_supported`, `:265`), and a cycle performs four of them — statistic-support probe, statistic values, basic values, flags. Descriptors are fixed for the life of the module, so with 2 descriptor pages that is 4 × 256 bytes of pure re-read. Caching at insertion (invalidated on removal) drops the bank from **1,530 → 506 bytes**, but only **192 → 184 transactions**.
2. **Block-read the VDM value pages.** Real values are fetched **two bytes at a time, one read per observable per lane** (`cmisVDM.py:112`) — 132 reads for 264 bytes, all contiguous inside pages `24h`/`25h`. Two 128-byte page reads carry the same data. This is the transaction win: **192 → ~62 per bank** on its own.

Both together: ~**500 bytes / ~56 transactions** per bank — roughly **3×** on bytes and **3.4×** on transactions. Note that (1) on its own barely moves the constraint that actually binds (see the transaction note above); (2) is the one that matters on a muxed or mailbox-backed bus.

**Not in the table above:**

| Item | When | Cost |
|---|---|---|
| `get_transceiver_info_firmware_versions` | **every** DOM cycle (`dom/dom_mgr.py:350`) | CDB cmd `0100h` — few bytes, but CDB status-polling latency per port |
| `DomThermalInfoUpdateTask` | its own faster interval | 2 bytes/port |
| `get_transceiver_info` | insertion only | 447 bytes |
| `get_transceiver_threshold_info` | insertion only | 84 bytes |
| `get_transceiver_vdm_thresholds` | insertion only | 1,282 bytes |

Insertion is heavier than a poll. On a shelf-wide cold start all 17 MCs pay it at once, so budget bus arbitration for the insertion burst separately from steady-state polling.

### Scaling beyond one CMIS space

If an MC ever needs **>32 lanes** or hard isolation between engines — while still presenting "one physical I2C" to the host — there are two escapes:

1. **I2C mux inside the MC** (PCA9548-style): the host bus fans out to N downstream segments, each with its own `0xA0` CMIS module (one per optical engine). Host selects a channel, then talks standard CMIS. Most common real-world CPO approach.
2. **Aggregator MCU**: one controller presents a single CMIS module and multiplexes engines behind **CDB commands / vendor pages**. More custom, single address, no mux.

For **17 MCs**, if each MC has its own `0xA0`, the host side needs **17 bus segments** — typically a **top-level I2C mux** (one channel per MC), each channel a single-CMIS-module MC as above.

### How an MC maps to SONiC ports

**One MC → one SONiC (Ethernet) port — the client side.** Attach the **line-side** parameters to that *same* port; do **not** create a second Ethernet `PORT` for the line side. Rationale:

- **No line-side port model in this tree.** The only OTN objects shipped are open-line-system components — `sonic-optical-attenuator.yang` (VOA), `sonic-optical-amplifier.yang` (EDFA), `sonic-channel-monitor.yang` (OCM), `sonic-alarm.yang`. There is **no** `OPTICAL_CHANNEL` / `TERMINAL_DEVICE` / `LOGICAL_CHANNEL` model for a line-side port to populate.
- **The line side already has a home on the Ethernet `PORT`.** `sonic-port.yang` carries the coherent knobs (`tx_power`, `laser_freq`) directly on the port — the same **400G ZR pattern** (one pluggable = one Ethernet port; `CmisManagerTask` reads those `CONFIG_DB PORT` fields and pushes them into CMIS). See the [400G ZR case study](#case-study-how-400g-zr-support-landed).
- **CMIS/xcvrd is one-module-centric.** One I2C → one `0xA0` → one CMIS memory → **one `TRANSCEIVER_INFO` row keyed by logical port**. The line side is Bank 1 of the same memory, not a separate managed module.
- **The line side isn't ASIC-facing.** A SONiC `PORT` represents a host/ASIC SerDes datapath the switch forwards on; the MC line side faces the **DWDM fiber plant**, so it has no business being an Ethernet `PORT`.

**The one real multiplicity — client-side breakout** (orthogonal to line/client): if the client 1.6T is broken out, one MC maps to *several* Ethernet ports, all sharing the *same* transceiver (`subport` / `lanes` in `sonic-port.yang`, exactly like a QSFP-DD breakout). The line side stays one optical carrier regardless.

| Client provisioning | SONiC Ethernet ports per MC |
|---|---|
| 1 × 1.6T | 1 |
| 2 × 800G | 2 |
| 8 × 200G | 8 |

**Interface mapping:**

| Entity | SONiC object | Notes |
|--------|--------------|-------|
| MC (physical module) | 1 cage / physical-port index | 1 CMIS `0xA0`; 1 `TRANSCEIVER_INFO` row |
| Client side (host-facing) | 1+ Ethernet `PORT` (`EthernetX`) | breakout → multiple ports, shared xcvr |
| Line side (network-facing) | coherent leaves on the **client** `PORT` (`tx_power`, `laser_freq`, …) | **not** a separate `PORT` |
| L3 usage | `INTERFACE` / sub-interface on the client port(s) | standard |
| Line optics telemetry | (future) OCM/amp/VOA tables if an OLS model is added | today: DOM/VDM on the client port's transceiver tables |

**When two ports would be justified:** only if the MC exposed two independent *host-facing* Ethernet datapaths the ASIC must switch separately (not a transponder). A CPO line side facing DWDM is not that. If a full terminal-device / optical-channel YANG model is added later, the line side becomes an *optical-channel object* referenced by the client port — still not a second Ethernet `PORT`.

**Putting it together:**

```text
        ┌───────────────────────── one MC ─────────────────────────┐
1 I2C ──►  one CMIS memory @ 0xA0
           ├─ Bank 0 (lanes 0–7)   ── one optical side (e.g. client)
           └─ Bank 1 (lanes 8–15)  ── other optical side (e.g. line)
                         │
                         ▼
        SONiC: 1 TRANSCEIVER_INFO row (keyed by logical port)
               client side  → Ethernet PORT(s)  ← interface mapping lives here
               line side    → coherent leaves on that same PORT (tx_power/laser_freq)
                             (NOT a separate Ethernet PORT)
```

> **Open issue — one `Sfp` object can only ever see one bank.** The diagram above implies `xcvrd` reads both banks of the MC. Today it cannot. `CmisApi.NUM_CHANNELS` is hardcoded to **8** (`api/public/cmis.py:109`), and the bank is frozen when the API is built: `create_xcvr_api(bank=bank)` → `CmisMemMap(CmisCodes, bank=bank)` (`xcvr_api_factory.py:53-74`), with `SfpBase.__init__(self, bank=bank)` (`sfp_base.py:75`) making **one `Sfp` object = one bank**. Reaching lanes 8–15 requires a *second* `Sfp` at bank 1, which needs its own physical-port index — and `DomInfoUpdateTask` iterates `port_mapping.physical_to_logical`, so that index is only polled if a logical port maps to it. With the one-port model above, **`xcvrd` polls Bank 0 only and the line side is never read.**
>
> This is a genuine tension with "one MC → one Ethernet `PORT`", not just a byte-count detail. Resolve it deliberately, one of:
>
> 1. **Two `Sfp` objects, one Ethernet `PORT`** — expose bank 1 as its own physical-port index for DOM purposes while keeping a single Ethernet `PORT` for forwarding. Preserves the port model; costs a second full poll (~1.5 KB, ~192 transactions per cycle) and needs the DB keying for the line-side row worked out.
> 2. **Platform-side bank aggregation** — have `Sfp.read_eeprom()` present a bank-1 view under vendor-specific offsets so one API object surfaces both sides. Keeps one port and one poll, but is a Molex-only deviation from `sonic-otn` and hides lanes from generic tooling.
> 3. **Upstream multi-bank support** in `sonic_xcvr` (lane count driven by `BanksSupported` rather than a hardcoded 8). Correct long-term fix, largest scope — align with `sonic-otn` before attempting.
>
> Until one of these lands, treat line-side DOM/VDM as **unimplemented**, and do not quote shelf-wide footprint numbers that assume 2 banks/MC are actually being polled.

### `xcvrd` implications

- **No `xcvrd` change for the bus**: banking, mux channels, and `0xA0` selection all live behind the **platform API** (`Sfp.read_eeprom()` / page + bank selection), exactly like the [FPGA-mailbox path](#platform-layer-and-optics-hardware-access). `xcvrd` just reads CMIS pages.
- `CmisManagerTask` handles multi-lane datapaths, but **only 8 lanes per API object** — `NUM_CHANNELS` is hardcoded (`api/public/cmis.py:109`) and the bank is fixed at construction. A 16-lane MC does *not* transparently surface as one 16-lane module; see the bank caveat [above](#how-an-mc-maps-to-sonic-ports). Ensure the platform's `get_transceiver_info()` reports the correct lane counts and the MC's `type` string is in `CMIS_MODULE_TYPES` (see workflow below).
- **DOM polling** (`DomInfoUpdateTask`) costs **~1,530 bytes across ~192 EEPROM reads per 8-lane bank per cycle** — ~3.0 KB / ~384 transactions per 16-lane MC, ~52 KB / ~6,500 transactions on a 17-MC shelf. **VDM is ~93% of it**, and ~1 KB/bank is redundant re-reading of static VDM descriptor pages. Size the poll interval and mux/bus arbitration on the **transaction count**, not the byte count. Full breakdown: [Monitoring data footprint](#monitoring-data-footprint).
- **The `dom_polling = disabled` knob is the blunt lever.** If bus arbitration can't absorb the shelf-wide poll, disable per-port DOM (`get_dom_polling_from_config_db`, `dom/dom_mgr.py:74`) or raise `--dom_update_interval` (default **60 s**, `DEFAULT_DOM_INFO_UPDATE_PERIOD_SECS`) before adding platform-local caching hacks.

## Workflow: adding a new transceiver type (example MC-1.6Tb)

Treat **MC-1.6Tb** as a placeholder for a new Molex media-channel / pluggable module. End-to-end work spans **EEPROM decode**, **platform drivers**, **`xcvrd`**, **JSON policy**, and sometimes **SAI / SerDes**. When changing `sonic-buildimage`, align with `sonic-otn/sonic-buildimage` for the same paths unless Molex has a deliberate delta.

1. **Confirm module identity on the wire**
   - CMIS modules use `CmisApi` (and helpers under `sonic_platform_base.sonic_xcvr`) with the CMIS memory map.
   - **SFF-8024 identifier byte**: if the module introduces a new standardized identifier/abbreviation, extend `XCVR_IDENTIFIERS` / `XCVR_IDENTIFIER_ABBRV` in `sonic_platform_base/sonic_xcvr/codes/public/sff8024.py` using the values from the current SFF-8024 revision (don't invent codes).

2. **Platform / EEPROM path**
   - In the vendor `sonic_platform` package (`platform/<vendor>/...`), ensure `get_sfp()` returns an object whose EEPROM is read with the correct xcvr API (e.g. `CmisApi`).
   - `get_transceiver_info()` must populate the keys `xcvrd` expects (`type`, `type_abbrv_name`, vendor/media fields, lane counts, …), feeding `STATE_DB TRANSCEIVER_INFO`.
   - If optics are reached via **FPGA mailbox** (PCIe mmap → SPI → line-card I2C), implement `read_eeprom()` / presence / change events in platform code — see **[Platform layer and optics hardware access](#platform-layer-and-optics-hardware-access)** above. `xcvrd` stays unchanged.

3. **`xcvrd`: CMIS and the type string**
   - `CmisManagerTask` only runs its state machine for module types in `CMIS_MODULE_TYPES`. If `TRANSCEIVER_INFO` exposes a new `type` string for MC-1.6Tb, add it there (keep naming consistent with what the platform publishes, e.g. `OSFP` vs `OSFP-8X`):

```python
class CmisManagerTask(threading.Thread):

    CMIS_MAX_RETRIES     = 3
    CMIS_DEF_EXPIRED     = 60 # seconds, default expiration time
    CMIS_MODULE_TYPES    = ['QSFP-DD', 'QSFP_DD', 'OSFP', 'OSFP-8X', 'QSFP+C']
    CMIS_MAX_HOST_LANES    = 8
```

   - `SfpStateUpdateTask` watches `TRANSCEIVER_INFO` (incl. `type`); keep types stable across insert/remove so CMIS and media/DOM logic behave.

4. **Media and SerDes policy JSON**
   - `media_settings.json` (parsed by `media_settings_parser`): keys match vendor/media/lane-speed derived from the transceiver dict when `notify_media_setting` runs. Add sections so MC-1.6Tb resolves to the correct ASIC media settings per port/lane speed.
   - `optics_si_settings.json` (parsed by `optics_si_parser`): add entries for the same identity keys so host SerDes/SI tuning is applied. Paths typically live under the device `hwsku` / `platform.json` area.

5. **Host side: port speed, breakout, and SAI**
   - Port `speed` / breakout in `CONFIG_DB` must match the MC-1.6Tb link design (e.g. aggregate 1.6 Tb/s across lanes).
   - New lane rates or FEC modes require coordinated SAI / syncd / vendor-SDK updates outside `xcvrd`.

6. **Optional: YANG / CLI / telemetry**
   - New `STATE_DB` fields/enums → update YANG and `show`/`config` paths that consume them.
   - Extend transceiver-type-specific PM/telemetry collectors if MC-1.6Tb exposes new VDM pages or counters.

7. **Verification checklist**

| Step | What to verify |
|------|------------------|
| EEPROM | Raw pages decode; `type` / `type_abbrv_name` match expectations. |
| `xcvrd` | `TRANSCEIVER_INFO` populated; `CmisManagerTask` runs without stuck states (unless skipped on purpose). |
| Policy | `media_settings` / `optics_si` apply (logs from the parsers). |
| Link | Data path up with correct speed/FEC; DOM/VDM tables update if applicable. |
| Regression | Re-run `sonic-xcvrd` tests; exercise insert/remove and warm paths if you touch `deinit()` / DB cleanup. |

8. **Molex vs sonic-otn** — for shared SONiC trees, diff the same file in the `sonic-otn/sonic-buildimage` reference before editing the Molex tree; keep Molex-only changes (extra `CMIS_MODULE_TYPES` entries, MC-specific JSON) minimal and documented in the commit message.

## Case study: how 400G ZR support landed

History was inspected in `sonic-platform-common`, `sonic-platform-daemons`, and `sonic-yang-models`. The real 400G ZR story is **mostly CMIS + API + `xcvrd` timing/coherent controls**, *not* a new `CMIS_MODULE_TYPES` string (ZR modules still use existing CMIS form factors such as QSFP-DD / OSFP).

| Workflow step | 400G ZR evidence |
|----------------|------------------|
| 1–2 EEPROM / decode | `sonic-platform-common` `c8eceec` ("400zr initial support", #228): large additions to `cmis.py`, `c_cmis.py`, `cmisVDM.py`, `sff8024.py` (host/media interface and compliance codes), mem-maps, `xcvr_api_factory`, tests — the bulk of "new module family" decode and APIs (`is_coherent_module`, laser/power helpers, VDM, …). |
| 3 `xcvrd` / CMIS manager | `sonic-platform-daemons` `cc56367` (#270): `CmisManagerTask` reads `laser_freq` / `tx_power` from `CONFIG_DB PORT`, validates grid/power, calls `api.set_laser_freq` / `set_tx_power`, `force_cmis_reinit` on change; uses `api.is_coherent_module()` instead of matching a new name in `CMIS_MODULE_TYPES`. `4ea12cf` (#293): datapath init/deinit timeouts from EEPROM (`get_datapath_init_duration`, `DpInitPending` for CMIS 5.x). Later `3d35404` (#442): VDM robustness for 400ZR keys. |
| 4 JSON policy | Not the main thread in the earliest ZR PRs; ZR tuning may still appear in per-platform SKU `media_settings.json` / `optics_si_settings.json`. |
| 5 Host / SAI | Port speed and pipeline work elsewhere; minigraph work (`f9bfa47e8` / `743625c2b` in `sonic-buildimage`: "parse 400g zr port config from minigraph") shows chassis/config integration. |
| 6 YANG / CLI | `sonic-yang-models` `201792f3c` (#11053): `sonic-port.yang` leaves for coherent config (descriptions still reference 400G ZR). |

**Takeaway:** the heavy lift is in `sonic_platform_base` / `sonic_xcvr` (decode + CMIS APIs), then `xcvrd`'s `CmisManagerTask` for the state machine and module-specific behavior, then YANG for operator-facing `PORT` keys, plus follow-up VDM/link fixes and tests. Unlike a "new abbreviation" example (MC-1.6Tb), 400G ZR did **not** primarily add a string to `CMIS_MODULE_TYPES` — it extended coherent CMIS behavior and `CONFIG_DB PORT` fields. A new form-factor code in SFF-8024 would still use workflow step 1.
