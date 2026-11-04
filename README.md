# MSI GeForce RTX 4090 Gaming X Trio: NVML and NVAPI control reference

An independent description of how a host program reads and controls the MSI GeForce RTX 4090 Gaming X Trio on
Windows: which of NVIDIA's documented NVML calls work on this GeForce card under WDDM, which need administrator
rights, how the setters for fans, power limit, locked clocks, clock offsets, the acoustic limit and PowerMizer behave
and how they are undone, where the hot-spot and memory-junction temperatures and the measured fan speed come from
(NVAPI's undocumented entry points), how the two NVML fans map to the two NVAPI coolers, and what the card did when
its hot spot reached 102 °C.

| Part | How the host reaches it | Document |
|---|---|---|
| Identity, telemetry, fans, power, clocks, offsets, thresholds, PowerMizer | NVIDIA's NVML library (`nvml.dll`) and read-only NVAPI entry points (`nvapi64.dll`), through the display driver | [msi-rtx4090-gaming-x-trio.md](msi-rtx4090-gaming-x-trio.md) |
| RGB lighting | an MSI controller on the GPU's I²C bus, driven by OpenRGB | the same document, section 11 (out of scope; not described) |

Coverage per function and change history: [STATUS.md](STATUS.md).

## Why this exists

NVML is documented by NVIDIA, but its header documents many functions only for "fully supported" products and lists
GeForce cards as "Limited Support", so a programmer cannot tell from the documentation which calls work on a GeForce
card under Windows, which need elevation, or how the card answers. The hot-spot and memory-junction temperatures and
the fan tach are not in NVML at all. From 2026-10-21 to 2026-11-04 this card was read, and from 2026-10-27 also
written, by independently written code on one PC, first beside and then without the vendor software. This reference
writes down what the card and driver did, what the published sources say, and what is still unknown.

## Scope

- NVML and the read-only NVAPI calls on Windows 11 with driver 610.88: the calls, their results and return codes on
  this card, privileges, fans and the fan-to-cooler map, temperature sources, power limit, locked clocks, offsets,
  thresholds, PowerMizer, restore paths and persistence, and one hot-spot excursion under load.
- Not covered: the card's RGB controller beyond the fact that OpenRGB drives it, NVAPI setters (none was called),
  the VF curve, overclocking advice, the VBIOS and any firmware or VBIOS flashing (deliberately not described), and
  MSI's or NVIDIA's user interfaces.
- One unit was observed, with VBIOS `95.02.3c.40.b8` and driver 610.88. Other VBIOS or driver versions may differ.

## How to read the tags

Every statement carries one of these tags.

| Tag | Meaning |
|---|---|
| **OBSERVED** | seen on this card through NVML or NVAPI by independently written third-party code (not the vendor software), with the date; includes what other tools on the same PC (LibreHardwareMonitor, OpenRGB, MSI's software while installed) showed at the time |
| **HOST** | read on the PC without driving the card: Windows' device properties, the driver and VBIOS versions, the NVIDIA app's telemetry database, the vendor software's log and profile files |
| **METADATA** | read from NVIDIA's or MSI's installed DLLs without running them: export tables, version resources and printable strings |
| **PUBLISHED** | from NVIDIA's NVML header and documentation, or from an open-source project, which is named under Credits; where this card confirmed it, the statement says so |
| **INFERRED** | a conclusion drawn from the evidence, not read from any source; the reasoning is given |
| **UNKNOWN** | a gap, with a note on what would close it |

There was no bus capture: NVML and NVAPI are calls into the display driver, not traffic on a USB or other
capturable bus.

## Sessions

Times are local time of the observing PC (UTC-7).

| Date | What was done |
|---|---|
| 2026-10-18 | read-only study of the installed `nvml.dll` and `nvapi64.dll` (exports, versions), the NVIDIA app's telemetry database and the vendor software's logs and profiles (HOST, METADATA) |
| 2026-10-21 13:38 and 14:27 | first runs of the third-party code, unelevated, beside MSI Center: identity, telemetry, NVAPI attach; the fan hand-back refused with `NO_PERMISSION` |
| 2026-10-26 | unelevated read of every value, with MSI Center holding the fans at 32 % |
| 2026-10-27 | elevated, with MSI Center stopped for the final runs: each fan held at 60 % and 30 % and handed back (the fan-to-cooler map); power limit 400 W and back; graphics and memory clocks locked and reset; restore on stop |
| 2026-10-28 to 2026-11-04 | the third-party code as a Windows service, reading the card about once a second; both fans taken to 100 % and handed back 48 times during thermal emergencies of the host (all but two triggered by the CPU); the driver's default fan policy restored at every start and stop |
| 2026-11-02 | MSI Center and MSI Afterburner removed from the PC |
| 2026-11-03 14:03 to 14:12 | clock offsets, the acoustic limit and PowerMizer set and restored; offsets under a shader load, which drove the hot spot to 102.6 °C |

## Sources

1. The card's answers through NVML and NVAPI, called by independently written code on the owner's PC.
2. Windows' device properties, the installed NVIDIA DLLs (exports and versions), the NVIDIA app's telemetry database,
   and MSI Center's and MSI Afterburner's log and profile files, read as files while those programs were installed.
3. NVIDIA's NVML header and the projects listed under Credits.

No NVIDIA or MSI software was run for this work, and no vendor source code was seen.

## Credits

Projects are linked to their repositories; the facts were taken from their source trees as of October 2026.

- **NVIDIA** publishes NVML and its header `nvml.h` (here the copy in **[go-nvml](https://github.com/NVIDIA/go-nvml)**,
  `gen/nvml/nvml.h`, NVML 13.4.92): function signatures, structures and version macros, return codes, the
  "fully supported" and "Limited Support" product classes, privilege notes, deprecations, the fan-control warning, the
  persistence statements, the clock-event reason bits and the field ids.
- **[LibreHardwareMonitor](https://github.com/LibreHardwareMonitor/LibreHardwareMonitor)** (commit `e68a7ef`,
  `LibreHardwareMonitorLib/Interop/NvApi.cs`, `Interop/NvidiaML.cs`, `Hardware/Gpu/NvidiaGpu.cs`) published the NVAPI
  function ids and structure layouts used here, the RTX 40 hot-spot and memory-junction decode and its selection by
  name, the thermal-mask probe, the fan-cooler status and control calls, the voltage decode, and its NVML binding.
- **[NvAPIWrapper](https://github.com/falahati/NvAPIWrapper)** (`NvAPIWrapper/Native/Helpers/FunctionId.cs`) lists
  the NVAPI ids of the calls not used here (fan-cooler info, power and thermal policies, VF curve, I²C write,
  illumination).
- **[OpenRGB](https://gitlab.com/CalcProgrammer1/OpenRGB)** (`i2c_smbus/Windows/i2c_smbus_nvapi.cpp`,
  `Controllers/MSIGPUController/`) drives the card's RGB controller over NVAPI I²C; its I²C path takes no cross-process
  lock for that bus.
- **[PawnIO.Modules](https://github.com/namazso/PawnIO.Modules)** (`Nvidia.p`) published which NVIDIA chips its
  hot-spot and memory-temperature reads accept.
- **NVIDIA** makes the GPU, the driver, NVML and NVAPI; **MSI** makes the card. Product names are theirs.

Nothing from these sources is copied here; only the facts are used, with credit. Function names, ids, constants and
structure sizes they publish are stated with credit.

## Related references

The same PC's other devices are documented in the same way, each in its own repository:

- [msi-meg-z790-ace-protocol](https://github.com/immortalbob/msi-meg-z790-ace-protocol): MSI MEG Z790 ACE motherboard: Mystic Light, the NCT6687D fan and sensor controller, MSI software and BIOS
- [msi-coreliquid-s360-protocol](https://github.com/immortalbob/msi-coreliquid-s360-protocol): MSI MEG CoreLiquid S360 cooler: controller and LCD
- [lian-li-uni-fan-sl-v2-protocol](https://github.com/immortalbob/lian-li-uni-fan-sl-v2-protocol): Lian Li UNI FAN SL V2 and the UNI HUB SL V2 controller
- [lian-li-universal-screen-protocol](https://github.com/immortalbob/lian-li-universal-screen-protocol): Lian Li Universal Screen 8.8": LCD and LED ring
- [razer-peripherals-protocol](https://github.com/immortalbob/razer-peripherals-protocol): Razer DeathAdder V2, Huntsman V2, Kraken V3 Pro and Synapse 3
- [asus-xg34wcdg-protocol](https://github.com/immortalbob/asus-xg34wcdg-protocol): ASUS ROG Strix OLED XG34WCDG monitor: DDC/CI and the Aura USB interface
- [msi-optix-g27c2-protocol](https://github.com/immortalbob/msi-optix-g27c2-protocol): MSI Optix G27C2 monitor: DDC/CI and EDID

## Licence

The documents in this folder are released under the **Creative Commons Attribution 4.0 International licence
(CC BY 4.0)**: use, copy, translate and build on them for any purpose, with credit to this work. See `LICENSE`.
No scripts are published with this reference.

## How to check your own card

Any language that can call a C library can repeat these reads: NVIDIA's `nvidia-smi` covers much of NVML from the
command line, and LibreHardwareMonitor shows the NVAPI hot spot, memory junction and fan tach.

1. Read first: identity, temperatures, fan duty and the NVAPI cooler list. Note your VBIOS and driver versions.
2. Before any fan write, make sure no other fan tool runs (MSI Center, Afterburner, LibreHardwareMonitor with fan
   control on), and run elevated.
3. Hold one fan at a time and watch which NVAPI cooler's tach follows; then hand it back with
   `nvmlDeviceSetDefaultFanSpeed_v2` and read the policy and the tach.
4. Change one setting at a time, read it back, and restore it before the next.

Contributions especially welcome: whether a fan hold survives the death of the process that set it, the results of
the NVML getters marked UNKNOWN, the power-limit and offset ranges, and results from other VBIOS versions. See
[STATUS.md](STATUS.md).

## Safety

A manual fan setting makes the caller responsible for cooling the card; NVIDIA's header warns that too low a setting
can damage the GPU. Apart from the calls listed in the document, nothing here was tested; anyone sending commands to
their own card does so at their own risk. Keep a hand-back path that works when the controlling program dies, watch
the hot spot and memory junction as well as the core, and do not write the card's RGB controller or its VBIOS from
your own code.
