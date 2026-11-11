---
name: sdr-tools-powerline
description: "SDR tools and RF sensing for powerline monitoring."
version: 0.1.0
author: ChrisL2000, Hermes Agent
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [SDR, RF, NILM, EMI, BLE, powerline, smart-home]
    related_skills: [openhue, grafana-observability-ops]
---

# SDR Tools + Powerline Monitoring Skill

Consolidated project for using software-defined radio and RF sensing against a
residential powerline service. Two threads:

- (A) NILM + EMI appliance monitoring — metering AFE (current/voltage harmonic
  fingerprints, the METER) fused with an SDR (radiated + conducted EMI, the
  DETECTOR). Full design in `references/spec.md`.
- (B) RF band coverage and Bluetooth/BLE tooling — what a given SDR can and
  cannot hear, and open monitor-mode BLE sniffing. Full detail in
  `references/rf-tools.md`.

## When to Use

- Resuming this project after a break.
- Adding appliance-level energy or EMI monitoring to a home lab.
- Deciding which SDR covers a target band (RTL vs HackRF vs Pluto vs sniffer).
- Setting up open monitor-mode BLE sniffing.

Don't use for: plain smart-plug on/off control (use `openhue` / Tasmota), or
generic Grafana dashboarding (use `grafana-observability-ops`).

## Locked Design Decisions (thread A)

- AFE: ADE9000-class metering front-end (7 ADC, Class 0.2, on-chip DSP, 32 kS/s
  waveform buffer resampled to 128 pt/cycle for >=50-harmonic FFT) on
  ESP32/STM32. Budget path: ATM90E32 / ADE9153A / MCP39F511.
- Isolation is mandatory on every signal path: clamp CT + burden for current;
  magnetic (non-switching) AC step-down transformer for the voltage reference;
  isolated coupling network for conducted EMI. Nothing touches mains directly.
- Architecture: on-device/host compute -> feature vectors, NOT raw continuous
  streaming (6 circuits @ 1 Hz = ~0.9 kB/s).
- SDR receiver tiers: characterize with RTL-SDR V3 first; upgrade to RSP1A/B or
  Airspy HF+ Discovery only if conducted signatures prove discriminating.
- Two-evening calibration runs FIRST and gates the receiver upgrade + permanent
  install. See `references/spec.md` section 8.

## Locked RF/BLE facts (thread B)

- Standard RTL dongles (R820T2/R860) cover ~24 MHz-1.7 GHz — they do NOT reach
  2.4 GHz (BLE/Wi-Fi) or 5/6 GHz. Full coverage table in `references/rf-tools.md`.
- Bluetooth Classic, LE, and Core 6.0/6.3/7.0 all run in the 2.4 GHz ISM band
  (2.402-2.480 GHz). There is no shipped "BLE 6" band; the 5/6 GHz BT work is an
  in-development spec project only.
- Open monitor-mode BLE sniffing = nRF52840 Dongle + Nordic nRF Sniffer firmware
  + Wireshark. External antenna = Fanstel BT840XE/BU840XE (nRF52840 + SMA).
  Native BLE 6.0 silicon (nRF54) has no official sniffer support yet.

## Prerequisites

- Host: any Linux box; Raspberry Pi 4 (2 GB) is the floor (SDR load is single-
  digit % of one core, ~300-600 MB RAM).
- SDR: RTL-SDR V3 + shielded near-field probe set (primary). Optional discone.
- BLE: nRF52840 Dongle (or Fanstel BT840XE for external antenna).
- Software: SoapySDR (+ backends), GNU Radio or scipy, Wireshark/tshark +
  nrfutil + nRF Sniffer plugin. Downstream: MQTT -> InfluxDB -> Grafana.

## Procedure (thread A build order)

1. Two-evening calibration — radiated first, zero mains work (see spec section 8).
2. Tier-1 conducted coupling network (only if Evening 2 needed) — BOM + safety
   in `references/spec.md` section 5.
3. AFE board (ADE9000-class) — spec after calibration justifies the build.
4. Host: calibration on the existing box; dedicated Pi only for a permanent
   install (RF PSU isolation + reboot decoupling).

## Quick Reference (verified numbers)

- Coupling cap 47 nF: 56 kOhm @ 60 Hz / 22.6 Ohm @ 150 kHz / 3.4 Ohm @ 1 MHz;
  ~2.1 mA leakage @ 120 V; use 33 nF @ 230 V.
- Sampling -> harmonic ceiling @ 60 Hz: 6 kS/s -> 50th, 25 kS/s -> 200th,
  100 kS/s -> 800th. Sweet spot = 50th harmonic (6-25 kS/s).
- Feature vector: scalars 32 B/ch, 2-50 harmonic vector 392 B/ch.
- SDR input ceiling: +10 dBm = 0.71 Vrms into 50 Ohm.
- RTL-SDR raw I/Q: V3 4.8 MB/s; RSP1A 12.1 MB/s; Airspy HF+ 3.1 MB/s.

## Pitfalls

- A 50/60 Hz isolation transformer is NOT an HF coupler (leakage inductance +
  winding capacitance kill 150 kHz-30 MHz); use a class-Y cap + earthed box.
- ADC dynamic range beats sample rate for RF fingerprinting (8/14/18-bit ->
  48/84/110 dB).
- Self-interference: board/host PSU radiates into the bands you listen to.
- RF is a presence/likelihood feature, not metrology — re-baseline per geometry.
- Single CT sized for 200 A cannot resolve a 50 mA charger.
- RTL-SDR direct-sampling mode is weak + has AM bleed-through in the low band;
  drive verdicts off tuner-mode 30 MHz-1 GHz.
- BLE sniffing is near-field by design: place the sniffer between the two
  devices; external antenna is for mounting distance, not gain.

## Verification

- Calibration table produced and saved (spec section 8).
- Coupling network: leakage < 5 mA, -61 dB mains remnant, clamp + 20 dB pad,
  box and host earthed before mains work.
- Feature vectors flowing into MQTT -> InfluxDB -> Grafana at Kbps, not raw.
- BLE sniffer visible in Wireshark via extcap; decodes advertising + connection
  link-layer traffic.

## References

- `references/spec.md` — v0.1 NILM/EMI powerline specification (single source
  of truth for thread A).
- `references/rf-tools.md` — SDR frequency coverage table + Bluetooth/BLE bands
  and open sniffer hardware (thread B).
