# SONiC Transceiver Management — Operator Guide

How to manage and monitor pluggable transceivers on a SONiC device. This guide is for **operators / NMS users**. You do **not** need to understand CMIS internals — SONiC abstracts the module form factor (SFP/QSFP/QSFP-DD/OSFP) behind a single set of CLI and APIs. For internals (architecture, the CMIS datapath state machine, adding a new module type), see the companion **[xcvrd Developer Guide](./xcvrd-developer-guide.md)**.

## What `xcvrd` does for you

`xcvrd` is SONiC's **transceiver daemon**. It detects insert/remove, reads the module EEPROM and live diagnostics, brings up CMIS modules, and **publishes everything into Redis `STATE_DB`**. You consume that data through CLI, gNMI telemetry, or SNMP — the daemon itself has no interface you talk to directly.

Key point: **CMIS form-factor details are handled internally by `xcvrd`. You use the same CLI regardless of module type.**

## Background: DOM, VDM, and status monitoring

These three terms are different levels of health/performance monitoring for optical modules.

### DOM (Digital Optical Monitoring)

Often called simply **diagnostics**; real-time **physical-layer vitals**:

- **Transceiver temperature** — internal module temperature
- **Supply voltage** — power delivered to the transceiver
- **TX bias current** — laser drive current (elevated current can indicate aging)
- **TX output power** — transmitted optical power
- **RX received power** — received optical power from the far end

### VDM (Versatile Diagnostic Monitoring)

An evolution of DOM on **newer CMIS modules** (QSFP-DD, OSFP). Deeper insight:

- **Pre-FEC BER** — signal quality *before* error correction
- **SNR** — signal-to-noise ratio
- **Histogram data** — some implementations expose statistics over time
- **Coherent metrics** — for 400G-ZR-class modules: dispersion, optical SNR, etc.

### Status

The **operational state** of the transceiver (distinct from DOM/VDM telemetry):

- **Module state** — initialized, low-power, data-path ready, etc.
- **Interrupts / flags** — hardware alarms or warnings (high temperature, loss of signal)
- **Presence** — whether the platform detects a module in the port

## CLI

### Show transceiver data (`show interfaces transceiver …`)

Read-only; reads `STATE_DB`. Works identically across all module form factors.

| Subcommand | Shows |
|---|---|
| `eeprom [-d/--dom]` | Vendor/module info from `TRANSCEIVER_INFO` (+ DOM) |
| `info` | `TRANSCEIVER_INFO` |
| `status` | Operational status (`TRANSCEIVER_STATUS`) |
| `pm` | Performance monitoring (`TRANSCEIVER_PM`) |
| `lpmode` | Low-power mode state |
| `presence` | Whether a module is present |
| `error-status [--fetch-from-hardware]` | Error state (`TRANSCEIVER_STATUS` / live HW) |

### Configure transceivers (`config interface transceiver …`)

- `lpmode <Ethernet#> <on/off>` — enable/disable module low-power mode
- `reset <Ethernet#>` — reset the module
- `dom <Ethernet#> <enable/disable>` — turn DOM polling on/off for the port
- `frequency <Ethernet#> <GHz>` — **coherent / 400G-ZR only**: set channel frequency
- `tx_power <Ethernet#> <dBm>` — **coherent / 400G-ZR only**: set Tx laser power

> Coherent (`frequency` / `tx_power`) tuning requires optical-domain knowledge (channel plan, power budget) — but that is *optical* knowledge, not CMIS internals.

### `sfputil` (lower-level tool)

`sfputil` talks **directly to the platform** (bypassing the DB) and is handy for bring-up/debug:
`sfputil show {eeprom,presence,error-status,lpmode,fwversion}`, `sfputil reset`, `sfputil lpmode on/off`, `sfputil power`, `sfputil firmware {version,download,run,commit,…}`.

## Northbound APIs

### gNMI / gNOI (streaming telemetry)

The primary programmatic interface. Clients subscribe/get `STATE_DB` paths directly, e.g. `STATE_DB/TRANSCEIVER_INFO/Ethernet0`, `…/TRANSCEIVER_DOM_SENSOR/Ethernet0`, `…/TRANSCEIVER_PM/…`. This is how most NMS/telemetry collectors stream DOM/PM data off-box.

### YANG / OpenConfig (REST + gNMI-translib)

OpenConfig transceiver models are present (`openconfig-platform-transceiver.yang` et al.), mapping data to `/components/component/transceiver` and the optical physical channels (laser bias, input/output power, temperature). Note: in this tree the db-backed gNMI path above is the more reliable way to retrieve OC transceiver/DOM data than translib REST.

### SNMP

Transceiver DOM is exposed as physical entities/sensors via ENTITY-MIB (`rfc2737`) and ENTITY-SENSOR-MIB (`rfc3433`), fed from `TRANSCEIVER_INFO` / `TRANSCEIVER_DOM_SENSOR`. Per-port temperature/voltage/Tx-bias/Rx-power appear as sensor OIDs.

## Troubleshooting tips

- **Module not detected:** `show interfaces transceiver presence`; if absent, check seating / `sfputil show presence`.
- **No DOM values:** confirm DOM polling isn't disabled (`config interface transceiver dom <port> enable`).
- **Link won't come up on a CMIS module:** `show interfaces transceiver status` shows a `cmis_state` field. If it never reaches `READY`, that's a signal to escalate — the CMIS bring-up sequence itself is internal (see the developer guide).
- **Errors:** `show interfaces transceiver error-status` for decoded error flags.
