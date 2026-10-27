# Temperature and cell-voltage registers, per brand

Survey of every protocol PDF we hold, for the long-term recording of ambient /
room temperature, battery pack and cell temperatures (min / avg / max) and cell
voltages. Findings are properties of the hardware and its published protocol,
not of any installation.

## Summary

| Brand / device | Ambient | Battery / BMS temp | Cell voltage | Status |
| --- | --- | --- | --- | --- |
| ATESS HPS / PCS / HPSTL / PBD | yes (36) | yes (167, 170, 171) | yes (174, 175) | implemented |
| Kehua BCS | no | no | no | nothing available |
| Sungrow inverter | no | no | no | nothing available |
| Sungrow Logger1000 | no | no | no | nothing available |
| Sungrow Logger3000 / 4000 | PT100 probe (8027, 8028) | no | no | implemented, gated by model |
| GoodWe GT / HT inverter | no | no | no | nothing available |
| GoodWe EzLogger | weather station only | no | no | needs a weather station + a new device class |

The short version: ATESS is the only brand in this repo that puts battery data
on the wire. The grid-tied PV inverters (GoodWe GT/HT, Sungrow) have no battery
and no BMS, so they publish an internal/heatsink temperature and nothing else —
there is no register to wire up. Kehua's storage converter talks to a BMS but
its protocol stops at voltage, current, SOC, SOH and limits; it exposes no pack
or cell temperature.

The `sunsynk/` folder under the protocol PDFs is reference material only. There
is no Sunsynk add-on in this repo or elsewhere in the org, and Sunsynk is out of
scope for this work.

## ATESS — ATESS Modbus RTU V3.22

Input registers. Already implemented before this ticket:

| Reg | Name | Type | Scale | Models |
| --- | --- | --- | --- | --- |
| 36 | Ambient temperature | 2 byte | 0.1 °C | all |
| 35 | Transformer temperature | 2 byte | 0.1 °C | HPS/PCS |
| 114 | PV module temperature | 2 byte | 0.1 °C | PBD |
| 171 hi / lo | BMS max / min temperature | 1 byte each | 1 °C | all |
| 174 / 175 | Max / min cell voltage | 2 byte | 0.001 V | all |
| 168, 172, 173 | Max/min cell-voltage group, unit and box numbers | 1 byte each | 1 | all |

Added in ATESS 0.5.8:

| Reg | Name | Type | Models |
| --- | --- | --- | --- |
| 167 hi / lo | BMS max / min temperature **group** number | 1 byte each | all |
| 170 hi / lo | BMS max / min temperature **box** number | 1 byte each | all |

Those four locate the 171 readings — which box and which group inside it ran
hottest and coldest — so a hot pack can be identified, not just detected. The
voltage equivalents (168, 172, 173) were already present; this closes the gap.

ATESS gives min and max only. There is no average and no per-cell array in the
protocol; an average would have to be computed downstream, and it would be an
average of two samples, so it is not worth publishing.

Not implemented, deliberately:

- 30–34 converter heatsink, buck fin, inductance and DC radiator temperatures,
  140 SCR temperature. Power-electronics temperatures, not battery or ambient —
  outside this ticket, cheap to add if derating investigations need them.
- 147 "BAT mode temperature" and 148 "balance board mode temperature". The
  applicable-model column in the PDF reads as a merged cell spanning rows
  144–148 for a single PCS model, and the English name is ambiguous ("mode" is
  likely "module"). If it is a battery module temperature it is exactly what
  this ticket wants, so it is worth one read against a PCS unit to confirm the
  value is plausible before adding it.

## Sungrow

**Inverter** (residential and commercial PV grid-connected protocol V1.1.53):
`Internal Temperature` at 5008 only, already implemented. The device has no
battery, so there is no BMS, cell or ambient register. The two "ambient
temperature" hits in the PDF are fault codes 37 and 43, not measurements.

**Logger** (logger protocol 1.0.2.9): PT100-1 at 8027 and PT100-2 at 8028,
S16, 0.1 °C. These are the only true environment-temperature registers Sungrow
offers us, and the protocol marks them Logger3000/Logger4000 only — a
Logger1000 has no PT100 input. Added in Sungrow 0.5.4 and dropped at connect
time for any other model.

Note the add-on's `supported_models` is still `Logger1000` alone, so these
registers stay inert until a Logger3000/4000 is enabled in
`sungrow_logger.py`. A Logger1000 site cannot read a temperature probe over
Modbus at all; it would need a meteo station on RS485 as a separate device.

The Logger1000 user manual documents a meteo-station input (RS485 or AI port,
`TEMPER_SENSOR` among the supported models) but gives no register table for it
— that is in the meteo station's own protocol document, which we do not hold.

## GoodWe

**GT and HT inverters**: both protocol documents contain exactly one
temperature register, the internal/chamber temperature (HT 32087, S16, 0.1 °C).
Everything else matching "temp" is an over-temperature fault or derating bit.
No battery, no BMS, no cell data.

**EzLogger**: has a weather station register block with
`Photovoltaic module temperature` at offset 2 and `Ambient temperature` at
offset 3, both S16 with gain 10 (so 0.1 °C). This is the only ambient reading
available anywhere in the GoodWe stack, and it comes with two conditions:

1. A Modbus weather station must physically be on the logger's RS485 bus.
2. Addressing is per-device. Either read it as its own logical device, using
   the weather station's RS485 address as the Modbus slave id and offsets
   0–24 as the register addresses; or read it through the logger's mapped
   register window at `51000 + 25 * (device address - 1) + offset`.

Wiring this up means a new `GoodweWeatherStation` server class (overriding
`read_model` and `is_available`, which both assume a `Device type code`
register the weather station does not have) plus the site's weather-station
address in config. Not built — it is speculative until we know a site has the
hardware.

## Kehua — Three-phase Energy Storage Converter protocol 004

The register map runs 5001–5211 and ends there. It carries `Inner temperature`
(5018) and `Radiator temperature` (5019), both implemented, and a BMS block at
5200–5211: system status, total battery voltage and current, SOC, SOH, charge
and discharge limits for current and voltage, and available charge/discharge
energy. No pack temperature, no cell temperature, no cell voltage, no ambient.
Everything the protocol offers is already implemented.

Getting battery temperatures off a Kehua site means talking to the battery BMS
directly rather than through the converter.

## Paneltrack

An AC power meter. No temperature of any kind. Out of scope.

## Sunsynk — out of scope

The Sunsynk 3-phase hybrid PDFs are reference material only; there is no
Sunsynk add-on to wire anything into. Recorded here solely so the next reader
does not re-survey the document: it is the only protocol we hold that publishes
per-pack battery temperature as average, cell-max and cell-min together with
per-pack max and min cell voltage. If a Sunsynk add-on is ever written, that
block is worth revisiting.

## Suggested next steps

1. Confirm on a PCS unit whether ATESS 147 is a battery module temperature; add
   it if so.
2. For GoodWe sites, establish whether any has a weather station on the logger
   before building the device class.
3. For ambient/room temperature on Kehua, Sungrow Logger1000 and GoodWe sites,
   accept that Modbus cannot supply it and record it from a separate sensor.
