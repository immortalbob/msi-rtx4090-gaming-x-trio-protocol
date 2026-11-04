# Status and change log

Part of the [MSI GeForce RTX 4090 Gaming X Trio reference](README.md). One row per function or area; the tags are
explained in the README. Driver 610.88, VBIOS `95.02.3c.40.b8`.

## NVML

| Function / area | Read | Write | Elevation | Open |
|---|---|---|---|---|
| init, identity, PCI info `_v3`, UUID, VBIOS, driver version | yes (OBSERVED) | | none | |
| serial | no usable value (OBSERVED) | | | the exact return code |
| core temperature (`GetTemperatureV` sensor 0) | yes (OBSERVED) | | none | |
| `GPU_MAX` sensor, field 82, T.Limit fields, margin | read, results not recorded | | | all (open item 3) |
| fan count, duty (`GetFanSpeed_v2`) | 2 fans; intended duty, 30 % reads 32 % (OBSERVED) | | none | |
| fan RPM (`GetFanSpeedRPM`), fan policy getter | not recorded | | | whether they answer |
| `SetFanSpeed_v2` | | **yes**, 30 / 60 / 100 % (OBSERVED 2026-10-27 to 2026-11-03) | elevated used; unelevated not tried | unelevated result; hold after process death |
| `SetDefaultFanSpeed_v2` | | **yes**; fans stop at idle afterwards (OBSERVED) | **required**: `NO_PERMISSION` unelevated (OBSERVED 2026-10-21) | |
| `SetFanControlPolicy` | | not needed | | return code |
| power usage, limit, enforced limit | yes (OBSERVED) | | none | constraints' values |
| `SetPowerManagementLimit_v2` | | **yes**, 400 W and 450 W; enforced limit lags one read (OBSERVED 2026-10-27) | elevated | behaviour under load at a reduced limit; reboot |
| locked graphics / memory clocks and resets | supported lists answer (INFERRED) | **yes**, 1500-2505 MHz and 10501 MHz (OBSERVED 2026-10-27) | elevated | the full supported lists |
| `Get/SetClockOffsets` (P0) | yes | **yes**, +15 graphics, +50 memory, live clocks moved (OBSERVED 2026-11-03) | elevated | P0 range; other P-states; persistence |
| acoustic threshold | 65 / 84 / 88 °C (OBSERVED) | **yes**, 80 and 84 °C (OBSERVED 2026-11-03) | elevated | effect on the fans |
| PowerMizer | yes | **yes**, prefer maximum performance and adaptive (OBSERVED 2026-11-03) | elevated | supported-modes mask |
| clock-event reasons | yes; none set under load (OBSERVED) | | none | state at a 102 °C hot spot |
| deprecated VF offsets, application clocks, driver model, persistence mode | | not tried | | |

## NVAPI (read-only)

| Function / area | Result | Open |
|---|---|---|
| initialise, enumerate, name, PCI ids, bus id | yes, elevated and not (OBSERVED) | |
| thermal sensors | mask `0x000000FF`; hot spot `T[1]/256`, memory junction `T[7]/256` plausible (OBSERVED) | cross-check with an independent tool; one hot-spot reading below the core |
| fan coolers status | two coolers, ids 1 and 2; measured RPM; NVML fan 0 = cooler 1, fan 1 = cooler 2 (OBSERVED 2026-10-27) | physical fans per channel |
| volt rails | answers (OBSERVED) | values |
| fan-cooler info (`GetInfo`) | fails on this card in MSI's software (HOST) | |
| any setter, I²C, illumination | **not called** | everything |

## Behaviour

| Area | Finding | Open |
|---|---|---|
| hot spot 102.6 °C under load, 2026-11-03 | no error, no reset, offsets kept; the host's guards took the fans to 100 % and handed them back (OBSERVED) | the card's own limit and slowdown |
| competing writers | MSI Center overwrote an NVML fan duty within about a second (OBSERVED) | |
| Windows service, sleep | NVML and NVAPI work from session 0 and survive S3 (OBSERVED) | held settings across sleep |
| persistence | power limit and locked clocks reset on reboot (PUBLISHED) | offsets, fan hold, acoustic limit, PowerMizer |
| RGB controller | OpenRGB drives it (OBSERVED) | **not described** |
| VBIOS and firmware | | **deliberately not described** |

## Change log

- **2026-11-04, first edition.** Written from NVML and NVAPI calls by independently written code on one PC
  (2026-10-21 to 2026-11-04: unelevated reads, elevated fan, power-limit and locked-clock tests on 2026-10-27, a
  Windows service reading the card for a week with thermal-emergency fan holds, and the offset, acoustic-limit and
  PowerMizer tests with a hot-spot excursion on 2026-11-03), the export tables and versions of the installed NVIDIA
  DLLs, Windows' device properties, the NVIDIA app's telemetry, MSI Center's and MSI Afterburner's logs and profiles,
  and the public sources credited in the README. New compared with what was public before: which "fully supported"
  NVML setters and getters work on this GeForce card under WDDM; the `NO_PERMISSION` answer of the fan hand-back
  unelevated; NVAPI cooler ids 1 and 2 and the fan-to-cooler map; the thermal mask `0x000000FF`; the idle fan stop
  under the driver's default policy; the enforced power limit's one-read lag; the locked-clock snapping; the memory
  offset appearing at half its value in NVML's memory clock; PowerMizer's effect at idle; the hot-spot excursion to
  102.6 °C with the core at 83 °C.
