# RF Band Coverage + BLE Tooling Reference

Companion reference for thread B of the sdr-tools-powerline project. Facts
verified against vendor datasheets / spec docs (Nov 2026).

## SDR / receiver frequency coverage

| Device | Freq range | ADC | Instantaneous BW | Notes |
|---|---|---|---|---|
| RTL-SDR V3 (R820T2/R860) | 24 MHz-1.766 GHz (practical ~1.7 GHz); direct-sampling 500 kHz-28.8 MHz | 8-bit | 2.4 MS/s (3.2 max) | ~$25. Does NOT reach 2.4 GHz. |
| RSP1A/B (SDRplay) | 1 kHz-2 GHz | 14-bit native <=6.048 MSPS (12b/10b/8b above) | up to 10 MHz | ~$120. Stops at 2 GHz. |
| Airspy HF+ Discovery | 500 Hz-31 MHz + 60-260 MHz | 18-bit DDC (22-bit narrow) | 768 kHz HF | US$169. 110 dB BDR. Gap 260 MHz-1 GHz. |
| Airspy R2 / Mini | 24 MHz-1.7 GHz | 12-bit | up to 10 MS/s | Same 1.7 GHz ceiling as RTL. |
| HackRF One | 1 MHz-6 GHz | 8-bit | 20 MHz | ~$300. Covers 2.4 GHz; half-duplex TX. |
| ADALM-PLUTO | 325 MHz-3.8 GHz | 12-bit | ~20 MHz | ~$250. Covers 2.4 GHz. |
| E4000-tuner RTL | 64-1700 MHz + 2.3-2.7 GHz | 8-bit | 2.4 MS/s | Discontinued/rare; the one RTL variant that reaches 2.4 GHz. |

Key takeaway: a standard RTL dongle tops out ~1.7 GHz, so it cannot see BLE
(2.4 GHz), Wi-Fi, or 5/6 GHz. For 2.4 GHz use HackRF One or ADALM-PLUTO. For
decoding BLE frames (not just seeing RF energy), use a sniffer dongle, not an SDR.

## Bluetooth / BLE frequency facts

- Bluetooth Classic, Low Energy, and Core spec 6.0 / 6.3 / 7.0 ALL operate in the
  2.4 GHz ISM band: 2.402-2.480 GHz (2.400-2.4835 GHz with guard bands).
- BLE uses 40 channels, 2 MHz wide, 3 advertising + 37 data; Classic uses 79
  channels, 1 MHz wide; Channel Sounding uses 79 x 1 MHz.
- Bluetooth 6.0 (Aug 2024) added Channel Sounding (PBR + RTT) — same 2.4 GHz PHY,
  no new band. 6.3 (May 2026) likewise 2.4 GHz.
- 5/6 GHz Bluetooth operation is a spec-development project ("Higher Bands for
  Low Energy" subgroup), NOT an adopted standard. Nothing to cover there yet.
- Implication: "BLE 6" traffic is capturable at the PHY by BLE 5.x silicon
  (the 1M/2M PHY is unchanged); only Channel Sounding's internal ranging math is
  opaque to older sniffers.

## Open monitor-mode BLE sniffing (recommendation)

- Primary: nRF52840 Dongle (PCA10059), ~$12-15. Flash with open-source `nrfutil`
  (`sniffer_nrf52840dongle` firmware), run `nrfutil ble-sniffer bootstrap` to
  register the Wireshark extcap, decode with Wireshark/tshark. Headless-friendly.
  Caveat: integrated PCB antenna, no external RF port.
- External antenna: Fanstel BT840XE / BU840XE — nRF52840 remote dongle shipping
  with an SMA-to-u.FL external antenna, wall-mount bracket, 2 m USB lead, same
  Nordic DFU bootloader (flashes the identical sniffer firmware). FCC/CE certified.
- DIY module path: Raytac MDBT50Q-U1MV2 or u-blox BMD-341/BMD-345 (u.FL), Laird
  BL654-Ant (u.FL + PA) on a USB carrier.
- Native BLE 6.0 silicon (nRF54L15): no official sniffer support as of Nov 2026
  (Nordic states no nRF54 sniffer support; third-party capture is scanner-only,
  advertising, no connection following, UART-limited). Avoid for now.
- Fully open-hardware alternative: Ubertooth One (open HW + firmware, SMA) — but
  BR/EDR-focused and weak for BLE; not recommended for BLE 6.

## Practical notes

- BLE sniffing is near-field by design: place the sniffer between the two
  communicating devices so it hears both sides of a connection.
- External antenna matters for mounting distance / rack placement, not link
  budget — the PCB-antenna dongle on a short USB extension is usually sufficient.
