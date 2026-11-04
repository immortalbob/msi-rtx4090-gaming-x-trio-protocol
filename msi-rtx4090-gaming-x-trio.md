# MSI GeForce RTX 4090 Gaming X Trio: NVML and NVAPI control on Windows (PCI `10DE:2684`, subsystem `1462:5103`)

Part of the [MSI GeForce RTX 4090 Gaming X Trio reference](README.md). Tags (**OBSERVED**, **HOST**, **METADATA**,
**PUBLISHED**, **INFERRED**, **UNKNOWN**) are explained there. Times are local time of the observing PC (UTC-7).
"The header" is `nvml.h` as NVIDIA publishes it in its go-nvml repository (NVML 13.4.92, the Linux CUDA
redistributable); it is cited by function name, not reproduced. NVML return codes are written with their
`nvmlReturn_t` name and number, for example `NVML_ERROR_NO_PERMISSION` (4). "Elevated" means a process with
administrator rights, including a Windows service running as LocalSystem.

Short version (each point is tagged in the sections below): the card has no USB or other host-visible control
device of its own; everything goes through the NVIDIA display driver. NVIDIA's documented NVML library gives
identity, the core temperature, fan duty, power, clocks, offsets, the acoustic temperature limit and PowerMizer, and
on this GeForce card under WDDM every NVML setter that was tried succeeded from an elevated process, including the ones
the header documents only for "fully supported" (non-GeForce) products. The hand-back of the fans to the driver's
default policy is refused with `NO_PERMISSION` from an unelevated process, although the header names no privilege
for it. The hot-spot and memory-junction temperatures and the measured fan tach exist only through NVAPI's
undocumented entry points, read the way LibreHardwareMonitor reads them. NVML has two fans, NVAPI has two coolers
numbered 1 and 2, and NVML fan 0 drives cooler 1. Under a sustained shader load the hot spot reached 102.6 °C while
the core read 83 °C; the card raised no error and the host's own guards handled it. Nothing the host sets is stored
on the card.

## 1. Device

### 1.1 Identity

| Item | Value | Tag |
|---|---|---|
| Card | MSI GeForce RTX 4090 Gaming X Trio | HOST (the subsystem id below; OpenRGB names the card so, section 11) |
| Host control path | the NVIDIA display driver only; the PC's USB device list holds no device belonging to the card | HOST |
| GPU | AD102, Ada architecture (`NVML_DEVICE_ARCH_ADA` = 8) | HOST (NVIDIA app telemetry) |
| PCI ids | vendor `10DE`, device `2684`, revision `A1`; subsystem `1462:5103` (MSI) | HOST (Windows device properties, NVIDIA app telemetry, MSI Center's log) |
| NVML name | `NVIDIA GeForce RTX 4090` | OBSERVED at every start, 2026-10-21 to 2026-11-04 |
| VBIOS | `95.02.3c.40.b8` | HOST; the same string from `nvmlDeviceGetVbiosVersion` (OBSERVED, every start) |
| Display driver | 610.88 (`32.0.16.1088`) | HOST; `nvmlSystemGetDriverVersion` returned `610.88` (OBSERVED) |
| Memory | 24 GB GDDR6X, 384-bit | HOST (MSI Center's log: 24564 MiB, memory type 15) |
| CUDA cores | 16384 | HOST (MSI Center's log) |
| Board power limit | 450 W default | HOST; reset to 450 W read back (OBSERVED 2026-10-27, section 6) |
| PCIe link | Gen 4, 8 lanes negotiated (the card is x16) | HOST (NVIDIA app telemetry). Whether the slot or a downtrain causes x8 is UNKNOWN |
| Fans | NVML: 2 fans; NVAPI: 2 coolers | OBSERVED at every start (section 5.1) |
| Serial number | none usable through NVML | OBSERVED: the observing code fell back to the GPU's UUID as its key; MSI's own serial query failed with NVAPI status −104 (`NotSupported`) (HOST, MSI Center's log) |

The NVIDIA app's telemetry database recorded a fan count of 0 for this card (HOST), while NVML reports 2 (OBSERVED).
The app's figure is not NVML's.

### 1.2 Software on the observed PC

| Item | Finding | Tag |
|---|---|---|
| `nvml.dll` | in `System32`, FileDescription "NVIDIA Management Library 610.88", FileVersion `8.17.16.1088`; 415 exports | METADATA |
| `nvml.dll` exports missing from the 13.4.92 header's list | `nvmlDeviceGetMemoryLimits_v1` / `SetMemoryLimits_v1`, `nvmlDeviceGetAdaptiveTgpModeInfo_v1` / `SetAdaptiveTgpMode_v1`, `nvmlDeviceGetBankRemapperStatus_v1`, `nvmlEventSetWait_v3`, the GPU-operational-event context calls (`nvmlEventSetGetContext*`, `nvmlEventSetRegisterGpuOperationalEvents_v1`), `nvmlDevicePerfMetricsGetSamples_v1`, `nvmlDeviceGetNvLinkTelemetrySamples_v1`, `nvmlDeviceGetGpuFabricInfo_v4`, `nvmlDeviceGetBBXTimeData_v1`, `nvmlDeviceSetNvlinkBwModeAsync_v1` | METADATA (export table against the header) |
| `nvml.dll` exports not in the header | `nvmlGetBlacklistDeviceCount`, `nvmlGetBlacklistDeviceInfoByIndex`, `nvmlInternalGetExportTable` (private; never bind) | METADATA |
| `nvapi64.dll` | in `System32`, version `32.0.16.1088`; exports only `nvapi_QueryInterface` and `nvapi_Direct_GetMethod` | METADATA |
| NVIDIA app | installed; its overclocking scanner and its automatic tuning were off; no user or factory overclock recorded | HOST (its settings database) |
| MSI Center (with its graphics-card engine) and MSI Afterburner | present until 2026-11-02, then removed; neither was used to control the card in this work | HOST |

Every function the rest of this document uses is present in this `nvml.dll` (METADATA).

## 2. Control paths

### 2.1 NVML

- **Library.** `nvml.dll` in `System32` (METADATA). NVIDIA's installer did not create an `NVSMI` folder on this PC,
  which LibreHardwareMonitor tries as a fallback location (HOST; PUBLISHED for the fallback). Load it by absolute path
  so that DLL search order cannot substitute another file (INFERRED, general Windows practice).
- **Binding.** The header maps unversioned names to `_v2` / `_v3` by macro (PUBLISHED). A program that binds exports
  by name must ask for the exact versioned name (`nvmlInit_v2`, `nvmlDeviceGetPciInfo_v3`,
  `nvmlDeviceSetFanSpeed_v2`) or it gets the legacy entry point (INFERRED from the macros). LibreHardwareMonitor binds
  seven exports this way, and its PCI-info structure is the legacy 52-byte one for `nvmlDeviceGetPciInfo_v2`, not
  the 68-byte `_v3` layout (PUBLISHED).
- **Calls.** C ABI, cdecl, every call returns an `nvmlReturn_t` (PUBLISHED). There are no packets, report IDs or
  byte order questions beyond the structures in section 12.
- **No exclusivity.** NVML talks to the display driver; there is no device handle to claim, and any number of
  programs can call it at once (PUBLISHED: the header says NVML is thread-safe; OBSERVED: the observing code,
  LibreHardwareMonitor and OpenRGB used the card at the same time for days). Conflicts are logical: the last writer
  of a setting wins (section 2.5).
- **Handle.** NVML's device index order is not guaranteed across reboots (PUBLISHED). Find the card by UUID or by PCI
  bus id, and confirm by `pciDeviceId = 0x268410DE` and `pciSubSystemId = 0x51031462` (section 14). On the observed PC
  the card is bus 1, device 0, function 0 (HOST).
- **Session 0.** NVML and NVAPI both work from a Windows service in session 0 (OBSERVED from 2026-10-28, every
  start of the service), unlike the monitor DDC/CI path on the same PC.
- **Sleep.** Across an S3 sleep on 2026-10-30 at about 06:40 and several overnight sleeps, the NVML session and its
  device handle kept working without a reconnect or an error (OBSERVED, by the absence of any NVML error or
  re-initialisation in the logs).
- **One stall.** On 2026-10-30 at 06:34:38 one NVML poll took longer than 5 s, together with LibreHardwareMonitor's
  polls of the CPU and the GPU; the next polls were normal (OBSERVED). The cause is UNKNOWN. Bound every NVML call
  with a timeout on a worker thread (INFERRED).

### 2.2 NVAPI (undocumented entry points)

`nvapi64.dll` exports only `nvapi_QueryInterface`; every function is fetched by a 32-bit function id and called
through the returned cdecl pointer, and every structure starts with a version word `size | (version << 16)`
(PUBLISHED, LibreHardwareMonitor). NVIDIA does not document these ids. The ids below are LibreHardwareMonitor's
(PUBLISHED); they returned data on this card (OBSERVED, every start from 2026-10-21, elevated and not elevated):

| Function | Id | What this card returned |
|---|---|---|
| `NvAPI_Initialize` | `0x0150E828` | OK |
| `NvAPI_EnumPhysicalGPUs` | `0xE5AC921F` | one GPU |
| `NvAPI_GPU_GetFullName` | `0xCEEE8E9F` | a name that selects LibreHardwareMonitor's RTX 40 decode (section 4.2) |
| `NvAPI_GPU_GetPCIIdentifiers` | `0x2DDFB66E` | the card's PCI ids; use them, not the index, to pair the NVAPI GPU with the NVML device |
| `NvAPI_GPU_GetBusId` | `0x1BE0B8E5` | the PCI bus number (1 on this PC, HOST) |
| `NvAPI_GPU_GetThermalSensors` | `0x65FE3AAD` | the hot spot and memory junction, sensor mask `0x000000FF` (section 4.2) |
| `NvAPI_GPU_ClientFanCoolersGetStatus` | `0x35AED5E8` | two coolers, ids 1 and 2, with RPM and level (section 5) |
| `NvAPI_GPU_ClientVoltRailsGetStatus` | `0x465F9BCF` | the core voltage (values not recorded) |

**No NVAPI setter, I²C call or illumination call was made in this work.** NVAPI fan control
(`NvAPI_GPU_SetCoolerLevels` `0x891FA0AE`, `NvAPI_GPU_ClientFanCoolersSetControl` `0xA58971A5`), power and thermal
policies, the VF curve and the I²C calls exist in LibreHardwareMonitor and NvAPIWrapper (PUBLISHED) and are
UNKNOWN on this card.

MSI's own software logged `NvAPI_GPU_ClientFanCoolersGetInfo() failed` hundreds of thousands of times on this card
(914 402 times with result −1, 2 171 with −220, 22 with −4) (HOST, MSI Center's logs). LibreHardwareMonitor never
calls `GetInfo` (PUBLISHED); the `GetStatus` call above works (OBSERVED). Avoid `GetInfo` (INFERRED). The meaning
of −220 is UNKNOWN; it is not in LibreHardwareMonitor's status list (PUBLISHED by absence).

### 2.3 Paths not used

- **PawnIO's NVIDIA module** reads GDDR6X per-chip temperatures through the GPU's MMIO on chip ids that include
  Ada's range, and the hot spot only on Blackwell (GB20x) chips (PUBLISHED, PawnIO.Modules). It needs a kernel driver
  and elevation and was not used here; its per-chip memory temperatures on this card are UNKNOWN.
- **The GPU's I²C bus** (NVAPI `I2CReadEx` / `I2CWriteEx`) carries the RGB controller (section 11). No program
  takes a cross-process lock for it: neither LibreHardwareMonitor nor OpenRGB does (PUBLISHED). Nothing was sent to
  it natively in this work.

### 2.4 Elevation

| Call | Unelevated | Elevated |
|---|---|---|
| NVML getters for identity, the core temperature and fan duty | answer (OBSERVED 2026-10-21 and 2026-10-26) | answer (OBSERVED, daily) |
| NVAPI getters of section 2.2 | answer (OBSERVED 2026-10-21 13:38, 2026-10-26) | answer (OBSERVED, daily) |
| `nvmlDeviceSetDefaultFanSpeed_v2` (hand-back) | **`NVML_ERROR_NO_PERMISSION` (4)** for both fans (OBSERVED 2026-10-21 13:38:20 and 14:27:42) | `NVML_SUCCESS` (OBSERVED, hundreds of times, section 5.4) |
| Every other setter of section 3 | not tried | `NVML_SUCCESS` (OBSERVED, section 3) |

The header documents no privilege and no `NO_PERMISSION` return for `nvmlDeviceSetDefaultFanSpeed_v2` and
`nvmlDeviceSetFanSpeed_v2` (PUBLISHED). On this card the hand-back needs elevation (OBSERVED), and the manual
setter almost certainly does too (INFERRED: the safer of the pair is already refused). In the unelevated runs the
observing code's fallback to `nvmlDeviceSetFanControlPolicy`, which the header marks "requires privileged user", may
also have run; its own return code was not recorded (UNKNOWN).

Consequence: a program that takes a fan must run elevated for as long as it holds it, and an unelevated helper
cannot release a hold left by an elevated one (INFERRED from the table).

### 2.5 Competing writers

- **MSI Center** held both fans at a manual 32 % at idle while it ran (OBSERVED 2026-10-26: NVML read 32 %, the
  tach 1072 and 1064 RPM), and it overwrote a fan duty set through NVML within about a second (OBSERVED 2026-10-27).
  Its graphics-card engine wrote clock offsets and the power cap through NVAPI (HOST: its log lists "Set Core
  Clock", "Set Power Cap: Power Limit(100)" and "Set Default Fan Speed (V3)" actions).
- **MSI Afterburner** kept a saved profile for this card with two fan channels (HOST). Its hardware layer names
  NVAPI internally (METADATA: strings `NVAPIHAL` in `RTHAL.dll`), so it is INFERRED to write through NVAPI as well.
- **LibreHardwareMonitor** stores each fan control's mode and value and re-applies them when it starts (PUBLISHED),
  so it can re-take a fan another program has handed back.
- **The NVIDIA app** had no overclock active (HOST).

None of them coordinates with any other: there is no lock, no notification and no owner field in NVML or NVAPI
(PUBLISHED by absence). Detect them by process and either stand back or ask the user to close them (INFERRED).
After 2026-11-02 only the observing code wrote fan, clock or power settings (HOST: the MSI software was removed).

## 3. What works on this GeForce card

The header lists GeForce cards as "Limited Support" and documents many functions only for "fully supported"
devices (Tesla, Quadro and similar) (PUBLISHED). For those functions the header says nothing about GeForce; this
table records what this card did with driver 610.88. "fs" = documented for fully supported devices only.

### 3.1 Setters

| Function | Header | This card, elevated |
|---|---|---|
| `nvmlDeviceSetFanSpeed_v2` | Maxwell and newer with fans; no privilege stated; switches the fan to manual (PUBLISHED) | `SUCCESS` at 60 %, 30 % and 100 % on each fan (OBSERVED 2026-10-27 and, at 100 %, 48 times per fan from 2026-10-30 to 2026-11-03) |
| `nvmlDeviceSetDefaultFanSpeed_v2` | products with fans; no privilege stated (PUBLISHED) | `SUCCESS` (OBSERVED, at every start and stop and after every hold); `NO_PERMISSION` unelevated (section 2.4) |
| `nvmlDeviceSetPowerManagementLimit_v2` | fs (Kepler+); requires root/admin; not persistent (PUBLISHED) | `SUCCESS`: 400 W, then 450 W (OBSERVED 2026-10-27, section 6). That the v2 entry point, not v1, carried the call is INFERRED from the observing code, which falls back to v1 only when v2 is missing |
| `nvmlDeviceSetGpuLockedClocks` / `nvmlDeviceResetGpuLockedClocks` | fs (Volta+); set requires root/admin (PUBLISHED) | `SUCCESS` for 1500-2505 MHz and the reset (OBSERVED 2026-10-27, section 7.1) |
| `nvmlDeviceSetMemoryLockedClocks` / `nvmlDeviceResetMemoryLockedClocks` | fs (Ampere+); set requires root/admin (PUBLISHED) | `SUCCESS` for 10501-10501 MHz and the reset (OBSERVED 2026-10-27) |
| `nvmlDeviceSetClockOffsets` | fs (Maxwell+); requires privileged user; out-of-range values are rejected with `INVALID_ARGUMENT`, not clamped (PUBLISHED) | `SUCCESS`: graphics and memory, P0, +15 / +50 MHz and back to 0, eight calls (OBSERVED 2026-11-03, section 7.2) |
| `nvmlDeviceSetTemperatureThreshold` (`ACOUSTIC_CURR`) | fs (Maxwell+); no privilege stated (PUBLISHED) | `SUCCESS`: 80 °C, then 84 °C, read back (OBSERVED 2026-11-03, section 4.3) |
| `nvmlDeviceSetPowerMizerMode_v1` | fs (Maxwell+); no privilege stated (PUBLISHED) | `SUCCESS`: prefer-maximum-performance, then adaptive (OBSERVED 2026-11-03, section 7.3) |
| `nvmlDeviceSetFanControlPolicy` | fs (Maxwell+); requires privileged user (PUBLISHED) | not needed elevated: no hand-back reported a fan still in manual (OBSERVED); its own result UNKNOWN |
| `nvmlDeviceSetGpcClkVfOffset` / `SetMemClkVfOffset` | deprecated in 13.0 (PUBLISHED) | not tried |
| application clocks (four calls) | deprecated in 13.0, to be removed in CUDA 14 (PUBLISHED) | not tried |
| `nvmlDeviceSetDriverModel`, `SetPersistenceMode` | driver model takes effect after a reboot and a display GPU must stay WDDM; persistence mode is "Linux only" (PUBLISHED) | not tried; never on a display card |

No setter returned `NOT_SUPPORTED`, and none caused `GPU_IS_LOST`, `RESET_REQUIRED` or a driver reset (OBSERVED,
2026-10-27 to 2026-11-04).

### 3.2 Getters

| Function | Header | This card |
|---|---|---|
| `nvmlInit_v2`, `nvmlDeviceGetCount_v2`, handles, `GetName`, `GetUUID`, `GetPciInfo_v3`, `GetVbiosVersion`, `nvmlSystemGetDriverVersion` | all products (PUBLISHED) | answer (OBSERVED) |
| `nvmlDeviceGetSerial` | products with an inforom (PUBLISHED) | no usable serial (OBSERVED, section 1.1); `NOT_SUPPORTED` is INFERRED |
| `nvmlDeviceGetTemperatureV` sensor 0 (`NVML_TEMPERATURE_GPU`) | all products (PUBLISHED) | answers (OBSERVED) |
| `nvmlDeviceGetNumFans`, `GetFanSpeed_v2` | products with fans (PUBLISHED) | 2; duty in percent (OBSERVED) |
| `nvmlDeviceGetMinMaxFanSpeed` | products with fans (PUBLISHED) | answers with a minimum of at most 30 % (OBSERVED: a 30 % floor was accepted); 30 / 100 per MSI's log (HOST) |
| `nvmlDeviceGetPowerUsage`, `GetPowerManagementLimit`, `GetEnforcedPowerLimit` | fs (PUBLISHED) | answer (OBSERVED: 77 W at idle in P0, 330-394 W under load; limits 400 and 450 W) |
| `nvmlDeviceGetPowerManagementLimitConstraints`, `GetPowerManagementDefaultLimit` | fs (PUBLISHED) | answer (INFERRED: the 400 W request was checked against them); values not recorded (UNKNOWN) |
| `nvmlDeviceGetClockInfo`, `GetPerformanceState` | fs (PUBLISHED) | answer (OBSERVED: 2595, 2730, 2745 MHz graphics; 10501, 10526 MHz memory; P0, P3, P8) |
| `nvmlDeviceGetSupportedGraphicsClocks`, `GetSupportedMemoryClocks` | fs (PUBLISHED) | answer (INFERRED from the snapped lock range, section 7.1) |
| `nvmlDeviceGetClockOffsets` | fs (PUBLISHED) | answers; read back +15 and +50 (OBSERVED 2026-11-03) |
| `nvmlDeviceGetTemperatureThreshold` acoustic min / current / max | fs (PUBLISHED) | 65 / 84 / 88 °C (OBSERVED 2026-10-21 and 2026-11-03) |
| `nvmlDeviceGetCurrentClocksEventReasons` | "all fully supported products" (PUBLISHED) | answers (OBSERVED: no reason set under the load of 2026-11-03) |
| `nvmlDeviceGetPowerMizerMode_v1` | fs (PUBLISHED) | answers (OBSERVED) |
| `nvmlDeviceGetFanSpeedRPM`, `GetFanControlPolicy_v2`, `GetMarginTemperature`, `GetTotalEnergyConsumption`, `GetTemperatureV` sensor 1 (`GPU_MAX`), field values 82 (`MEMORY_TEMP`) and 193-196 (T.Limit) | fs or unstated (PUBLISHED) | UNKNOWN: read by the observing code, results not recorded |

## 4. Temperatures

### 4.1 Sources

| Reading | Path | Tag |
|---|---|---|
| Core ("GPU") | `nvmlDeviceGetTemperatureV`, version `0x0100000C`, sensor 0; whole °C. The older `nvmlDeviceGetTemperature` is deprecated in 13.0 | PUBLISHED; OBSERVED |
| Hot spot | NVAPI `GetThermalSensors`, `Temperatures[1] / 256` °C | PUBLISHED (LibreHardwareMonitor's RTX 40 branch); OBSERVED plausible on this card (section 4.2) |
| Memory junction | NVAPI `GetThermalSensors`, `Temperatures[7] / 256` °C | as above |
| Hottest point of the die, NVML | `GetTemperatureV` sensor 1 (`NVML_TEMPERATURE_GPU_MAX`, "hottest part of the GPU die") | PUBLISHED; on this card UNKNOWN |
| Memory, NVML | field 82 `NVML_FI_DEV_MEMORY_TEMP` | PUBLISHED; on this card UNKNOWN |

NVML has no hot-spot or memory-junction getter that is documented for GeForce, and no hot-spot threshold
(PUBLISHED, by absence in the header).

### 4.2 The NVAPI thermal-sensor decode

LibreHardwareMonitor chooses the decode by the start of the marketing name, not by architecture: a name starting
`NVIDIA GeForce RTX 40` gives hot spot = `T[1] / 256` and memory junction = `T[7] / 256`; RTX 50 and others use
other indices (PUBLISHED). This card's name matches the RTX 40 branch (OBSERVED).

The sensor mask is found by probing: call with `1 << bit` for bit 0, 1, 2 and so on; at the first bit that fails,
use all lower bits (PUBLISHED, LibreHardwareMonitor). On this card bits 0 to 7 answer and bit 8 fails, so the mask
is **`0x000000FF`** and `T[7]` lies inside it (OBSERVED at every start, 2026-10-21 to 2026-11-04).

Values that fit the decode (OBSERVED):

| When | Core (NVML) | Hot spot `T[1]` | Memory junction `T[7]` |
|---|---|---|---|
| 2026-10-26, idle, fans held at 32 % by MSI Center | 35 °C | 43.8 °C | 40 °C |
| 2026-11-03 14:10:45, shader load, 330-394 W | 82 °C | 101.4 °C | 72 °C |
| 2026-11-03 14:11:07, after 22 s with every fan at 100 % | 77 °C | 94.8 °C | 68 °C |
| 2026-11-03 14:12:44, second excursion | 83 °C | 102.6 °C | 74 °C |
| 2026-11-03 14:12:51, load ending | 82 °C | 80 °C | 72 °C |

- Under load the hot spot ran about 19 to 20 °C above the core; at idle about 9 °C (OBSERVED, the rows above).
- LibreHardwareMonitor's library, running in the same host program with its own NVAPI calls, read 102.5 °C at
  14:12:44.203 against the observing code's 102.6 °C at 14:12:44.641 (OBSERVED). Both use the same decode, so this
  confirms consistency, not the decode itself (INFERRED). Comparison with an independent tool such as HWiNFO was not
  made (UNKNOWN).
- At 14:12:51 the hot spot read 80 °C with the core at 82 °C (OBSERVED). A hot spot below the core is not expected, since the hot
  spot is the hottest point of the die (INFERRED); probably the two values come from different calls a few hundred milliseconds apart while the load stopped. Open
  item 4.
- LibreHardwareMonitor notes that `NvAPI_GPU_GetThermalSettings` (a different call) returns a count of 0 when polled
  too fast (PUBLISHED). The observing code read `GetThermalSensors` about once a second for a week with no failure
  logged (OBSERVED).

### 4.3 Thresholds and the acoustic limit

- Acoustic thresholds through `nvmlDeviceGetTemperatureThreshold`: minimum 65 °C, current 84 °C, maximum 88 °C
  (OBSERVED: 84 °C read on 2026-10-21; "65..80..88" while 80 °C was set on 2026-11-03).
- `nvmlDeviceSetTemperatureThreshold(ACOUSTIC_CURR)` took 80 °C and then 84 °C, each read back at once (OBSERVED
  2026-11-03 14:03 to 14:04). What the acoustic limit changes in the fan behaviour of this card was not measured
  (UNKNOWN); the header describes `ACOUSTIC_CURR` only as the current acoustic threshold within its minimum and
  maximum (PUBLISHED).
- The slowdown and shutdown thresholds and the T.Limit fields 193 to 196 were read by the observing code but not
  recorded (UNKNOWN). The header says the T.Limit fields supersede the threshold getter on Ada (PUBLISHED); whether
  they read as absolute temperatures or relative to T.Limit on this card is UNKNOWN.

## 5. Fans

### 5.1 Fans and coolers

- **NVML** reports 2 fans, indices 0 and 1 (OBSERVED, every start).
- **NVAPI** `ClientFanCoolersGetStatus` reports 2 coolers whose **ids are 1 and 2**, not 0 and 1 (OBSERVED
  2026-10-27 onwards).
- **The map** (OBSERVED 2026-10-27, one fan held at a time with the other on the driver's policy):

| NVML fan | Commanded | NVAPI cooler that followed | Tach |
|---|---|---|---|
| 0 | 60 % | cooler **1** | 2059 RPM |
| 1 | 60 % | cooler **2** | 2069 RPM |

The cooler carries three fans (INFERRED from the product name; not counted for this document) and NVML has two
channels; MSI's own software also showed two fan groups, "Fan Percent" and "Fan Percent 2" (HOST, MSI Center's log).
Which physical fans share a channel was not checked (UNKNOWN).

NVML fan indices and NVAPI cooler ids are different numbering spaces; pair them by changing one and watching the
other, as above, not by assuming the same number (INFERRED; on this card they differ by one).

### 5.2 Intended and measured values

- `nvmlDeviceGetFanSpeed_v2` and `nvmlDeviceGetFanSpeedRPM` return the **intended** speed: the header says a blocked
  fan still reads its target (PUBLISHED). They cannot detect a stalled fan.
- NVAPI `ClientFanCoolersGetStatus` `CurrentRpm` moved with the commanded duty and fell to 0 at the driver's idle
  stop (OBSERVED 2026-10-27), so it is INFERRED to be the measured tach. LibreHardwareMonitor uses it as the fan
  sensor (PUBLISHED).
- Values seen (OBSERVED):

| State | NVML duty | Tach |
|---|---|---|
| MSI Center's manual hold, idle (2026-10-26) | 32 % | 1072 / 1064 RPM |
| 60 % commanded through NVML (2026-10-27) | | 2059 / 2069 RPM |
| 30 % commanded through NVML (2026-10-27) | **32 %** | not recorded |
| Driver's default policy, idle (2026-10-27) | | 0 RPM (fans stopped) |

A commanded 30 % reads back as 32 % (OBSERVED). INFERRED: the driver converts the percentage to its own fan scale and
back, so a read-back can differ from the command by a couple of points; compare with a tolerance.

### 5.3 Manual control

`nvmlDeviceSetFanSpeed_v2(device, fan, percent)` switches that fan's policy to manual and holds the duty until
something restores the default; the header warns in capitals that the caller then owns the card's cooling and can
damage it with too low a setting (PUBLISHED). On this card (OBSERVED, elevated):

- 60 % and 30 % on each fan took effect at once (2026-10-27).
- 100 % on both fans was accepted during every host thermal emergency, 48 times per fan between 2026-10-30 and
  2026-11-03, within 6 ms of the emergency in the two cases timed in section 8.
- The duty range the user can set is 30 to 100 % (HOST, MSI Center's log: "Fan Percent Min/Max: 30/100" for both
  groups; consistent with NVML accepting a 30 % floor, OBSERVED). Below the minimum the header returns
  `INVALID_ARGUMENT` (PUBLISHED); the observing code clamped requests to 30 % itself, so the card's answer to 10 %
  was not seen (UNKNOWN).

### 5.4 Handing a fan back

`nvmlDeviceSetDefaultFanSpeed_v2(device, fan)` restores the driver's default policy; the header names it as the undo
for the manual setter (PUBLISHED). On this card (OBSERVED, elevated):

- It returned `SUCCESS` every time: after each of the 48 emergency holds per fan, at every start of the observing
  code (the code hands both fans to the driver before it does anything else) and at every stop (2026-10-27 to
  2026-11-04).
- After the hand-back the fans **stopped at idle** (0 RPM) and started again with load: the VBIOS default curve
  includes MSI's idle fan stop (2026-10-27). No MSI software is needed for the idle stop.
- A fresh process can release the fans: on 2026-10-27 at 09:37:35 a separate watchdog process opened its own NVML
  session after the controlling process had died, and its hand-back returned `SUCCESS` (OBSERVED). Whether the
  hold survives the death of the process that set it, or the driver reverts by itself, was not tested (UNKNOWN, open
  item 1).
- Unelevated it is refused with `NO_PERMISSION` (section 2.4).
- A read-back of the policy with `nvmlDeviceGetFanControlPolicy_v2` never showed a fan still in manual after a
  successful hand-back (OBSERVED, by the absence of the observing code's warning); whether that getter answers on
  this GeForce card at all is UNKNOWN.

### 5.5 A caution about one published close-time restore

LibreHardwareMonitor addresses a fan control by the NVAPI `CoolerId`, but its close routine restores fans by loop
index `0, 1, ...` (PUBLISHED, `NvidiaGpu.cs` at commit `e68a7ef`). On this card the cooler ids are 1 and 2
(OBSERVED). INFERRED: if the restore matches coolers by id, index 0 matches nothing and cooler 2 is never restored;
if `SetCoolerLevels` takes a zero-based index instead, it may work. Which one happens was not tested (UNKNOWN). Any
host should restore the fans it actually set, by the ids or indices it used, and read the state back.

### 5.6 MSI's "Zero Frozr"

MSI's software exposes a fan-stop switch ("Zero Frozr") through its own graphics-card engine (METADATA: command
strings `CMD_VGA_Engine_ZeroFrozr_ON` / `_OFF` and an export `NDA_GetZeroFrozr` in MSI's DLLs). It is not an NVML
function (PUBLISHED, by absence), and how MSI implements it is UNKNOWN. The driver's default policy already stops the
fans at idle on this card (section 5.4).

## 6. Power limit

| Step (2026-10-27, elevated) | Result (OBSERVED) |
|---|---|
| set 400 W (`400000` mW) | `SUCCESS`; `nvmlDeviceGetPowerManagementLimit` read 400 W at once |
| `nvmlDeviceGetEnforcedPowerLimit` right after the set | still 450 W; 400 W on the next read (**the enforced limit lags one read**) |
| set back to 450 W | `SUCCESS`, read back 450 W |
| set 400 W, then the controlling process stopped with its normal restore | 450 W read afterwards by a separate read-only probe |

- The default and the value after every restore is 450 W (OBSERVED; HOST).
- The header says the limit does not persist across a reboot or a driver reload (PUBLISHED). A reboot test was not
  made (UNKNOWN).
- The minimum and maximum from `GetPowerManagementLimitConstraints` were not recorded (UNKNOWN; open item 5).
- A load under the 400 W cap was not run, so whether the card holds it and sets the software power-cap reason
  (`0x0004`) is UNKNOWN.
- For `nvmlDeviceSetPowerManagementLimit_v2` the structure is `{version, powerScope, powerValueMw}` with scope 0 = the
  GPU; the header's macro gives version `0x0200000C` although a comment in the structure says the version "must be
  1" (PUBLISHED). The macro value was accepted (INFERRED, section 3.1).
- The NVML power field values 185 to 192 need `scopeId` set, and their scope numbering (0 GPU, 1 module, 2 GPU base)
  differs from the power-value structure's (0 GPU, 1 module, 2 memory, 3 GPU base) (PUBLISHED).

## 7. Clocks

### 7.1 Locked clocks

| Request (2026-10-27, elevated) | Sent | Result (OBSERVED) |
|---|---|---|
| graphics 1500-2500 MHz | **1500-2505 MHz** | `SUCCESS`; the idle graphics clock rose to 1500 MHz |
| `nvmlDeviceResetGpuLockedClocks` | | `SUCCESS`, unlocked |
| memory 10501-10501 MHz | 10501-10501 | `SUCCESS`, held at 10501 MHz |
| `nvmlDeviceResetMemoryLockedClocks` | | `SUCCESS`, unlocked |

- Values must come from `nvmlDeviceGetSupportedGraphicsClocks` / `GetSupportedMemoryClocks` (PUBLISHED). The
  observing code moved each bound to a supported value, which turned 2500 into 2505 (OBSERVED). INFERRED: the
  supported graphics clocks on this card are spaced 15 MHz apart (1500 and 2505 are both multiples of 15); the full
  list was not recorded (UNKNOWN).
- The header allows the symbolic limits `0xFFFFFF01` (TDP) and `0xFFFFFF02` (unlimited), but both arguments must
  then be symbolic; a numeric and a symbolic value together are rejected (PUBLISHED). Only numeric values were sent.
- Locked clocks do not persist across a reboot or driver reload (PUBLISHED).
- No driver reset or GPU loss occurred (OBSERVED).

### 7.2 Clock offsets

`nvmlDeviceGetClockOffsets` / `nvmlDeviceSetClockOffsets` take a 24-byte structure `{version 0x01000018, type,
pstate, clockOffsetMHz, minClockOffsetMHz, maxClockOffsetMHz}`; type 0 = graphics, 2 = memory; pstate 0 = P0
(PUBLISHED). Out-of-range values are rejected, not clamped (PUBLISHED), so read the minimum and maximum for the same
type and P-state first.

| Time (2026-11-03) | Call | Result (OBSERVED) |
|---|---|---|
| 14:03:57.779 | graphics, P0, 0 → +15 MHz | `SUCCESS`, read back +15 |
| 14:04:06.427 | graphics, P0, +15 → 0 | `SUCCESS` |
| 14:04:12.042 | memory, P0, 0 → +50 MHz | `SUCCESS`, read back +50 |
| 14:04:20.634 | memory, P0, +50 → 0 | `SUCCESS` |
| 14:10:24.505 and 14:10:24.803 | graphics +15 and memory +50, together, under load | `SUCCESS` |
| 14:11:26.828 and 14:11:27.130 | both back to 0 | `SUCCESS` |

Under a sustained shader load (P0, 330-394 W, core 75-82 °C) the live clocks moved from 2730 to **2745 MHz**
(graphics, +15) and from 10501 to **10526 MHz** (memory) (OBSERVED 2026-11-03 14:10 to 14:11), and returned to
2730 / 10501 MHz after the reset.

- **The memory offset shows at half its value in NVML's memory clock.** +50 MHz moved `GetClockInfo(MEM)` by +25 MHz
  (OBSERVED). INFERRED: NVML's memory clock for this GDDR6X card is half the figure the offset is expressed in, so
  offsets and the reported memory clock are on different scales. Tools that show "21 Gbps" or "10501 MHz" for the
  same memory are using different multipliers (INFERRED).
- The offsets stayed applied while the host's thermal emergency took the fans (OBSERVED, section 8).
- MSI's software logged a core range of 1595 to 3595 MHz around a base of 2595 MHz (±1000) and a memory range of
  9501 to 13501 MHz around 10501 MHz (−1000 / +3000) (HOST, MSI Center's log). INFERRED: those are MSI's slider
  ranges, not the driver's limits. The driver's own minimum and maximum for P0 were not recorded (UNKNOWN; open
  item 6).
- Only P0 was changed. Whether other P-states have their own offsets that matter is UNKNOWN.
- Persistence of offsets across a reboot is not stated in the header (PUBLISHED, by absence) and was not tested
  (UNKNOWN).
- The deprecated `nvmlDeviceSetGpcClkVfOffset` / `SetMemClkVfOffset` reach the same setting by another path
  (PUBLISHED); using both paths on one domain has UNKNOWN effects.

### 7.3 PowerMizer

`nvmlDeviceGetPowerMizerMode_v1` / `nvmlDeviceSetPowerMizerMode_v1` use an unversioned 12-byte structure
`{currentMode (out), mode (in), supportedPowerMizerModes (out, bitmask)}`; modes 0 adaptive, 1 prefer maximum
performance, 2 auto, 3 prefer consistent performance (PUBLISHED).

- Prefer maximum performance: the idle card went to P0 at once, 2595 MHz graphics, 10501 MHz memory, 77 W
  (OBSERVED 2026-11-03 14:04).
- Adaptive: back to P3, then P8 (OBSERVED).
- The supported-modes mask was not recorded (UNKNOWN).

### 7.4 Clock-event reasons

`nvmlDeviceGetCurrentClocksEventReasons` returns a bitmask: `0x0001` GPU idle, `0x0004` software power cap, `0x0008`
hardware slowdown, `0x0020` software thermal slowdown, `0x0040` hardware thermal slowdown, `0x0080` hardware power
brake, among others (PUBLISHED). The three hardware bits are defined only under the older `nvmlClocksThrottleReason*`
names in the header (PUBLISHED). Under the shader load of 2026-11-03 at 2730 MHz the bench record notes no reason set
(OBSERVED); whether a bit was set at the exact moment of the 102 °C peak was not recorded (UNKNOWN).

## 8. The hot-spot excursion of 2026-11-03

The load was a browser-based GPU load page (a WebGL shader) in a GPU-accelerated browser. The host program on the
PC had a thermal emergency of its own at a hot spot of 102 °C, which sets every fan and pump of the PC to 100 %, and
the code that drove the card had a guard that forces the GPU fans it holds to maximum at a hot spot of 100 °C, with
5 °C hysteresis. Both are host software, not card features. Timeline (OBSERVED):

| Time | Event |
|---|---|
| 14:10:24.5 | core +15 and memory +50 MHz offsets applied (section 7.2) |
| 14:10:44.978 | hot spot 102 °C reaches the host's emergency limit |
| 14:10:44.983-.984 | `nvmlDeviceSetFanSpeed_v2(100)` on both fans: `SUCCESS` |
| 14:10:45.336 | the GPU guard trips: core 82 °C, hot spot 101.4 °C, memory junction 72 °C |
| 14:11:07.553 | guard released: core 77 °C, hot spot 94.8 °C, memory junction 68 °C |
| 14:11:12.261-.265 | emergency over; `nvmlDeviceSetDefaultFanSpeed_v2` on both fans: `SUCCESS` |
| 14:11:26.8-27.1 | offsets back to 0 |
| 14:12:44.203 | a second excursion: hot spot 102.5 °C (as LibreHardwareMonitor read it) triggers the emergency again |
| 14:12:44.206 | both fans to 100 %: `SUCCESS` |
| 14:12:44.641 | guard trips: core 83 °C, hot spot 102.6 °C, memory junction 74 °C |
| 14:12:51.718 | guard released (load ending): core 82 °C, hot spot 80 °C, memory junction 72 °C |
| 14:12:53.322-.323 | fans handed back: `SUCCESS` |

What the card did (OBSERVED):

- No NVML call failed, no `GPU_IS_LOST` or `RESET_REQUIRED`, no driver reset, no visible fault. The offsets stayed
  applied through the first excursion.
- With every fan at 100 % the hot spot fell 6.6 °C in 22 s under continued load.
- The core stayed at 82-83 °C. The guard's core ceiling, which it derives from the card's live slowdown limit minus
  a margin, read 94 °C, so the card's core slowdown limit lies at or above 94 °C (INFERRED); its exact value was not
  recorded (UNKNOWN).

What is not known: the card's own hot-spot limit (NVIDIA publishes none for GeForce; NVML has no hot-spot threshold,
PUBLISHED by absence), whether the card would have slowed itself at a higher hot spot, and why the hot spot runs
about 20 °C above the core under load on this unit (UNKNOWN). A host that controls the fans should watch the NVAPI
hot spot and memory junction, not only the NVML core temperature (INFERRED from the table: the core gave no warning).

## 9. Sequences

### 9.1 Start and read

1. `nvmlInit_v2` once per process; reference-count it if several parts of a program use NVML, with one
   `nvmlShutdown` per successful init (PUBLISHED: LibreHardwareMonitor serialises its NVML use the same way).
2. Find the device by UUID or PCI bus id; check `pciDeviceId = 0x268410DE`, `pciSubSystemId = 0x51031462`.
3. Read identity once; poll the getters of section 3.2 about once a second. Treat `NOT_SUPPORTED`,
   `FUNCTION_NOT_FOUND`, `DEPRECATED` (26) and `ARGUMENT_VERSION_MISMATCH` (25) as "absent" and stop asking (INFERRED).
4. NVAPI: `NvAPI_Initialize`, `EnumPhysicalGPUs`, pair the GPU by `GetPCIIdentifiers`, probe the thermal mask once
   (section 4.2), then read `GetThermalSensors`, `ClientFanCoolersGetStatus` and `ClientVoltRailsGetStatus` with the
   NVML poll. A hot spot of 0 means "unavailable"; do not substitute another sensor (INFERRED).
5. Before any write, check that no other fan or clock writer runs (section 2.5).

### 9.2 Hold a fan

1. Run elevated (section 2.4).
2. Record that the fan is held **before** the call, so a crash in between still restores it (INFERRED).
3. `nvmlDeviceSetFanSpeed_v2(fan, percent)` with percent clamped to the minimum and maximum (30 to 100 % here) and
   never below a safety floor.
4. Watch the core, the NVAPI hot spot and memory junction, and the NVAPI tach of the mapped cooler (section 5.1):
   0 RPM at a non-zero level for several seconds is a stall; force 100 % or hand back.
5. Re-assert the duty from time to time and re-check the read-back with a tolerance (section 5.2): another program
   may have overwritten it (section 2.5).

Steps 1, 3 and 5 are what the observing code did on this card (OBSERVED); the stall rule is INFERRED from
section 5.2.

### 9.3 Hand back and restore

In this order, each with its result checked (all OBSERVED to succeed on this card, elevated):

1. `nvmlDeviceSetDefaultFanSpeed_v2` on **every** fan, not only the ones this process remembers: a hold left by an
   earlier, crashed process is invisible to it (INFERRED).
2. Clock offsets back to the saved values with `nvmlDeviceSetClockOffsets`, per (type, P-state) changed.
3. `nvmlDeviceResetGpuLockedClocks`, `nvmlDeviceResetMemoryLockedClocks` if locked.
4. The power limit back to the saved value (450 W here) within the constraints.
5. The acoustic limit (84 °C here) and the PowerMizer mode (adaptive here), if changed.
6. `nvmlShutdown`.

Run the same undo on normal exit, on an unhandled exception and from an out-of-process watchdog, because a manual
fan hold may outlive the process (UNKNOWN, open item 1). A reboot resets the power limit and locked clocks
(PUBLISHED); for offsets and fan holds that is UNKNOWN.

## 10. Persistence and standalone behaviour

| Setting | Survives | Tag |
|---|---|---|
| Power limit | not a reboot or driver reload | PUBLISHED |
| Locked clocks | not a reboot or driver reload | PUBLISHED |
| Clock offsets | UNKNOWN (no statement in the header; not tested) | UNKNOWN |
| Manual fan hold | the controlling process's death: UNKNOWN; a reboot: INFERRED no (the card starts on its VBIOS curve) | UNKNOWN / INFERRED |
| Acoustic limit, PowerMizer | UNKNOWN | UNKNOWN |
| An S3 sleep | the NVML session survived; held settings across sleep were not checked | OBSERVED / UNKNOWN |

- **Without any host software** the card runs its VBIOS fan curve with idle fan stop (OBSERVED after hand-backs,
  section 5.4), at its default 450 W limit and stock clocks (INFERRED).
- **Self-protection** (hardware thermal slowdown, power brake, shutdown threshold) is enforced by the card whatever
  the host sets; NVML offers no call that disables it (PUBLISHED, by absence).
- **No flash.** No NVML or NVAPI call used here is documented to write the card's non-volatile memory; they are
  driver state (INFERRED from the reset-on-reboot statements and the absence of any save call).

## 11. RGB lighting (out of scope)

The card's lighting is an MSI controller on the GPU's own I²C bus, which OpenRGB detects through NVAPI I²C as
"MSI GeForce RTX 4090 Gaming X Trio" on "Nvidia NvAPI I2C on GPU 0", device address `0x68`, and drives with its
"MSI GPU v2" controller (PUBLISHED, OpenRGB; OBSERVED: OpenRGB 0.9+ listed it on this PC with one zone, and showed
a rainbow wave and "off" on it from 2026-10-31 to 2026-11-04 through its SDK server). While MSI Center ran, its
lighting service held the same device (OBSERVED). Nothing native was sent to this controller in this work,
and its protocol is not described here. OpenRGB's controller has a save function; whether it writes non-volatile
memory with limited endurance is UNKNOWN, so do not save in a loop (INFERRED).

The NVIDIA illumination API (`NvAPI_GPU_ClientIllumZones*`) is used by OpenRGB for a different controller type
(PUBLISHED); whether it does anything on this card is UNKNOWN.

## 12. Structure versions

NVML versioned structures carry `size | (version << 24)` in their first field; NVAPI ones `size | (version << 16)`
(PUBLISHED). A wrong word returns `NVML_ERROR_ARGUMENT_VERSION_MISMATCH` (25) or NVAPI `IncompatibleStructVersion`
(−9) (PUBLISHED). The sizes below were computed from the published layouts for x64 Windows (INFERRED, computed twice
with different tools); the ones marked "accepted" were used on this card with success (OBSERVED).

| Structure | Size | Version word | Note |
|---|---|---|---|
| `nvmlTemperature_v1_t` | 12 | `0x0100000C` | accepted |
| `nvmlClockOffset_v1_t` | 24 | `0x01000018` | accepted |
| `nvmlPowerValue_v2_t` | 12 | `0x0200000C` | `powerScope` is one byte at offset 4, `powerValueMw` at 8; accepted (INFERRED, section 6) |
| `nvmlDevicePowerMizerModes_v1_t` | 12 | none (unversioned) | accepted |
| `nvmlPciInfo_t` (for `_v3`) | 68 | none | `busId` at 36; LibreHardwareMonitor's 52-byte layout is the `_v2` one |
| `nvmlMemory_v2_t` | 40 | `0x02000028` | |
| `nvmlFieldValue_t` | 40 | none | value union at 32; for `valueType` 2 (`unsigned long`) read **4** bytes on Windows, not 8 |
| `nvmlFanSpeedInfo_v1_t` | 12 | `0x0100000C` | |
| NVAPI `NvThermalSensors` v2 | 168 | `0x000200A8` | `Temperatures[32]` at 40; accepted |
| NVAPI `NvFanCoolersStatus` v1 | 1704 | `0x000106A8` | 32 items at 40; accepted |
| NVAPI `NvGpuClientVoltRailsStatus` v1 | 76 | `0x0001004C` | core µV at `0x28` (low) and `0x2C` (high); accepted |

## 13. Hazards

| Hazard | Guard |
|---|---|
| A manual fan hold at a low duty, then a crash | The header warns the caller now owns cooling (PUBLISHED). Record the hold before the call; restore on every exit path and from a watchdog process; never go below the 30 % minimum; force 100 % on a thermal ceiling |
| Watching only the NVML core temperature | The hot spot reached 102.6 °C with the core at 83 °C (OBSERVED). Watch the NVAPI hot spot and memory junction too |
| Stall detection from NVML | NVML fan values are intended speeds (PUBLISHED). Use the NVAPI tach, allow for the idle stop (0 RPM at level 0) |
| Unelevated hand-back | Refused with `NO_PERMISSION` (OBSERVED). Keep elevation while anything is held |
| Assuming NVML fan index = NVAPI cooler id | They differ by one on this card (OBSERVED) |
| A close-time restore by loop index | LibreHardwareMonitor's may miss a cooler here (section 5.5) |
| Other writers (MSI Center, Afterburner, LibreHardwareMonitor) | Last writer wins, within a second for MSI Center (OBSERVED). Detect and stand back, or offer a monitor-only mode |
| Out-of-range offsets | Rejected, not clamped (PUBLISHED). Clamp to the live minimum and maximum; ramp in small steps and stop on `GPU_IS_LOST` |
| Mixed symbolic and numeric locked clocks | Rejected (PUBLISHED). Use values from the supported lists |
| `SetDriverModel`, persistence mode | Never on a display card; persistence mode is Linux only (PUBLISHED) |
| `NVML_ERROR_INSUFFICIENT_POWER` (8), `GPU_IS_LOST` (15) | Hardware alarms: stop writing and tell the user (PUBLISHED meanings; INFERRED handling) |
| NVAPI I²C beside OpenRGB or MSI's software | No cross-program lock exists (PUBLISHED). Leave the bus to one program |
| The RGB controller's save | Endurance UNKNOWN; never in a loop |
| Deprecated exports in a later driver | Bind by exact name and treat a missing export or `DEPRECATED` as "absent" |
| `nvmlInternalGetExportTable` | Private, undocumented; never call |

## 14. Test vectors

| Id | Input | Expected |
|---|---|---|
| T1 | `nvmlDeviceGetPciInfo_v3` | `pciDeviceId = 0x268410DE` (device `2684` in the high half, vendor `10DE` in the low half), `pciSubSystemId = 0x51031462`, bus 1 (OBSERVED / HOST) |
| T2 | `nvmlDeviceGetName`, `nvmlDeviceGetVbiosVersion` | `NVIDIA GeForce RTX 4090`, `95.02.3c.40.b8` (OBSERVED) |
| T3 | `nvmlDeviceGetNumFans`; NVAPI cooler count | 2; 2 with ids 1 and 2 (OBSERVED) |
| T4 | NVAPI thermal mask probe | bits 0-7 answer, bit 8 fails → mask `0x000000FF` (OBSERVED) |
| T5 | `T[1] = 26112` (`0x6600`), `T[7] = 18432` (`0x4800`) | hot spot 102.0 °C, memory junction 72.0 °C (arithmetic of the PUBLISHED decode) |
| T6 | volt-rails low `1050000`, high `0` | 1.05 V (arithmetic of the PUBLISHED decode) |
| T7 | `nvmlTemperature_v1_t` for the core, as bytes | `0C 00 00 01  00 00 00 00` + 4 output bytes |
| T8 | `nvmlClockOffset_v1_t`, memory, P0, +50 MHz, as bytes | `18 00 00 01  02 00 00 00  00 00 00 00  32 00 00 00` + 8 output bytes |
| T9 | `nvmlPowerValue_v2_t`, GPU scope, 400 W, as bytes | `0C 00 00 02  00 xx xx xx  80 1A 06 00` (`xx` = padding); 450 W: `... D0 DD 06 00` |
| T10 | `SetFanSpeed_v2(fan 0, 60)` | cooler **1** tach about 2060 RPM (OBSERVED 2059) |
| T11 | `SetFanSpeed_v2(fan 0, 30)`, then `GetFanSpeed_v2` | 32 (OBSERVED); accept a small difference |
| T12 | `SetDefaultFanSpeed_v2` unelevated | `NVML_ERROR_NO_PERMISSION` (4) (OBSERVED) |
| T13 | `SetGpuLockedClocks` request 1500-2500 after snapping to the supported list | 1500-2505 (OBSERVED) |
| T14 | power limit set to 400 W, then `GetEnforcedPowerLimit` at once and one poll later | 450 W, then 400 W (OBSERVED) |
| T15 | memory offset +50 MHz under load | NVML memory clock 10501 → 10526 MHz (OBSERVED) |
| T16 | PowerMizer prefer maximum performance at idle | P0, 2595 MHz, about 77 W (OBSERVED) |
| T17 | `GetTemperatureThreshold` acoustic min / current / max | 65 / 84 / 88 (OBSERVED) |
| T18 | anti-vectors | an NVML fan index used as a cooler id reads the wrong tach; an 8-byte read of a `valueType` 2 field value takes 4 bytes of garbage; the 52-byte PCI-info layout with `_v3` corrupts memory |

## 15. Open items

| # | Question | How to settle |
|---|---|---|
| 1 | Does a manual fan hold survive the death of the process that set it? | hold 60 % from an elevated process, kill it without cleanup, read the policy and the tach |
| 2 | Is `nvmlDeviceSetFanSpeed_v2` also refused unelevated, and what does `SetFanControlPolicy` return? | one call each from an unelevated process, ending with an elevated hand-back |
| 3 | What do `GetTemperatureV` sensor 1, field 82, the T.Limit fields 193-196, `GetMarginTemperature`, `GetFanSpeedRPM` and `GetFanControlPolicy_v2` return on this card? | a read-only probe logging each return code and value |
| 4 | Are NVAPI `T[1]` and `T[7]` really the hot spot and memory junction on this VBIOS, and why did the hot spot read below the core once? | compare with HWiNFO under a steady load and while the load stops |
| 5 | Power-limit minimum and maximum; behaviour under a reduced limit | read the constraints; run a load at 400 W and log draw and reasons |
| 6 | The driver's offset range for P0 and for other P-states | `GetClockOffsets` for each type and P-state |
| 7 | Do offsets, the acoustic limit or PowerMizer survive a reboot or a sleep? | set, reboot or sleep, read back |
| 8 | Which physical fans belong to NVML fan 0 and fan 1? | hold one channel, look at the card |
| 9 | What does the acoustic limit change in fan behaviour? | log duty and tach against temperature at two settings |
| 10 | The card's own hot-spot limit and slowdown behaviour | log the clock-event reasons every 100 ms during a hot-spot excursion |
| 11 | Why the link runs at x8 | check the slot wiring in the board manual and the BIOS lane settings |
| 12 | NVAPI fan control, power and thermal policies on this card | elevated tests of `ClientFanCoolersSetControl` and the policy getters, one at a time |
| 13 | What LibreHardwareMonitor's close-time restore does with cooler ids 1 and 2 | hold both coolers in LibreHardwareMonitor, close it, read the policies |
| 14 | Per-chip GDDR6X temperatures through PawnIO on AD102 | an elevated LibreHardwareMonitor run showing its GPU memory sensors |
