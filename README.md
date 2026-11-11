# sdr-tools-powerline

Software-defined radio tools and RF sensing for residential powerline
monitoring. A consolidated project covering two threads:

- **NILM + EMI appliance monitoring** — a metering AFE (current/voltage harmonic
  fingerprints, the *meter*) fused with an SDR (radiated + conducted EMI, the
  *detector*). Non-intrusive appliance fingerprinting on 120/220 V 60 Hz.
- **RF band coverage + BLE tooling** — what each SDR can and cannot hear, plus
  open monitor-mode Bluetooth Low Energy sniffing.

This repo is a **Hermes Agent skill** plus the project's design references. No
hardware is built yet.

## Contents

- `SKILL.md` — the loaded skill: locked decisions, procedure, verified numbers,
  pitfalls, verification.
- `references/spec.md` — v0.1 NILM/EMI powerline specification: isolation
  posture, AFE decision, bandwidth architecture, Tier-1 conducted coupling
  network BOM, SDR receiver tiers + host sizing, and the two-evening calibration.
- `references/rf-tools.md` — SDR frequency coverage table, Bluetooth/BLE band
  facts, and open monitor-mode BLE sniffer hardware (nRF52840 dongle / Fanstel).
- `README.md` — this file.

## Status

v0.1 — design + front-end spec locked. Awaiting hardware calibration. No mains
work done. The two-evening calibration gates the receiver upgrade and any
permanent install.

## Key facts (summary)

- A standard RTL dongle covers ~24 MHz-1.7 GHz — it cannot see BLE/Wi-Fi at
  2.4 GHz or 5/6 GHz.
- Bluetooth (Classic, LE, and Core 6.0/6.3/7.0) is still 2.4 GHz; the 5/6 GHz
  Bluetooth work is an in-development spec project, not shipped.
- Open monitor-mode BLE sniffing: nRF52840 Dongle + Nordic nRF Sniffer firmware
  + Wireshark. External antenna: Fanstel BT840XE/BU840XE.
- Isolation is mandatory on every mains-adjacent signal path (CT, transformer,
  coupling network). Nothing touches mains directly.

## Safety

This project involves residential mains. The isolation rules in
`references/spec.md` section 2 and the Tier-1 safety rules in section 5 are
non-negotiable. Treat any coupling-network output as live, earth box and host,
and verify against a published reference (Jim Brown K9YC "Power Line Noise and
RFI") before touching mains.

## License

MIT
