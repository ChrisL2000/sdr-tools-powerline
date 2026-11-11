# Residential NILM + SDR Appliance Monitor — Specification

Status: design + front-end spec (v0.1). Not built. No mains work done yet.

## 1. Objective
Non-intrusively monitor 120/220 V 60 Hz residential service and fingerprint
individual appliances via two fused modalities:

- (A) Current/voltage harmonic fingerprinting (AFE) — measures watts, RMS, pf,
  THD, harmonic vector. This is the METER (quantifies utilization).
- (B) Radiated + conducted EMI fingerprinting (SDR) — identifies WHICH appliance
  is running. This is a DETECTOR, not a meter (no power/energy).

## 2. Isolation posture (non-negotiable)
- Current: clamp CT (induction) + burden resistor. No conductor contact.
- Voltage reference: magnetic (non-switching) AC step-down transformer, e.g.
  120 V -> 9 V AC. Preserves phase, provides isolation. Never a direct tap or
  capacitive divider on mains.
- Conducted EMI: isolated coupling network (Section 5). Isolated but mains-coupled;
  requires care.
- Radiated EMI: loop antenna, zero mains coupling.

## 3. Sensing front-end decisions (locked)
- Device class: metering AFE (ADE9000) on ESP32/STM32 — on-device compute,
  7 ADC, Class 0.2, 32 kS/s waveform buffer, resampled 128 pt/cycle for >=50-harmonic
  FFT. SPI (20 MHz) to host MCU. Budget path: ATM90E32 / ADE9153A / MCP39F511.
- Turn-key alternative (utilization only, no harmonics): IoTaWatt (ADS1115 ~860 S/s
  — NOT fingerprint-grade).
- Raw-waveform lab captures: AD3 (USB bench) or LabJack T7 (Ethernet, 16-bit, 100 kS/s).
- Gating question carried: single-phase (2 ch: 1 CT + 1 Vref) vs split-phase 240 V
  (4 ch — second device or aggregate-only). OPEN.

## 4. Bandwidth / interface architecture (locked)
- Do NOT stream raw continuously. On-device/host compute -> feature vectors.
- Feature vector: scalars (RMS V, RMS I, P, Q, S, pf, phase, THD) = 32 B/ch; 2-50
  harmonic vector = 392 B/ch. 6 circuits @ 1 Hz = ~0.9 kB/s — any link.
- Raw waveform (bench/train): 2ch 16-bit @ 25 kS/s = 100 kB/s (50th harmonic);
  transients @ 100 kS/s = 400 kB/s burst-only (ring buffer + drain).
- Sampling->harmonic ceiling @ 60 Hz: 3 kS/s->25th, 6 kS/s->50th, 25 kS/s->200th,
  100 kS/s->800th. Sweet spot = 50th harmonic (6-25 kS/s).
- Interfaces: Ethernet (100 Mbps) for headless 24/7; WiFi ok for feature path;
  SPI on-board; USB for bench. SDR raw stays on-host (USB dongle -> host FFT).

## 5. SDR — conducted EMI coupling network (Tier 1)
Goal: safe, repeatable tap of 150 kHz-30 MHz conducted noise into a 50 Ohm SDR
input (max +10 dBm = 0.71 Vrms). Not a compliance LISN.

Topology (single-ended hot-to-earth, plug-in probe at one wall outlet):
  Hot -> 100 mA fuse -> 47 nF X2 safety cap -> 1 kOhm -> node
    node -> BAT54S clamp to earth
    node -> 20 dB 50 Ohm pad -> SMA center
    SMA shield = earth (box bonded to cord ground)
  Neutral NOT used as RF return (floats w.r.t. earth).

Why it works (computed):
- 47 nF: 56 kOhm @ 60 Hz (blocks mains, ~2.1 mA leakage @ 120 V = under GFCI 5 mA
  trip), 22.6 Ohm @ 150 kHz, 3.4 Ohm @ 1 MHz (passes noise). Acts as high-pass.
- 60 Hz remnant reaching pad ~ 106 mV (-61 dB). Transients still need clamp + pad.

BOM (Tier 1, ~$40-60):
- C1: 47 nF 275 VAC X2 safety cap, 15 mm pitch (TDK/EPCOS B32922, KEMET R46, WIMA MP3X2).
  NOTE: @ 230 V EU, 47 nF leaks ~4.1 mA (near RCD edge); use 33 nF there.
- R1: 1 kOhm 0.25 W (limits transient current into clamp).
- D1: BAT54S dual Schottky back-to-back to earth (or SMAJ5.0CA bidirectional TVS;
  ESD-grade low-C diode for less RF loading).
- ATT: 20 dB 50 Ohm Pi/Tee pad (or inline SMA fixed attenuator) — required, do not skip.
- F1: 100 mA 250 V fuse + holder (or fused IEC inlet).
- Enclosure: die-cast aluminum (Hammond 1590B); SMA (or BNC) bulkhead.
- Cord: 3-conductor 18 AWG power cord (or IEC C14 inlet + cord); earth bonded to box.
- Optional: series DC block (10-100 nF 50 V) at SMA.

Safety rules (read before touching mains):
- Treat the output as LIVE. Earth the box AND the host PC (3-prong) so SDR USB
  ground = earth = reference.
- Never rely on the coupling cap for touch safety; class-Y/X2 = fails open.
- Verify design against a published reference (Jim Brown K9YC "Power Line Noise and
  RFI" coupler) before building.
- Start at a single wall outlet (plug-in probe), NOT the panel.

## 6. SDR receiver (locked at tiers)
- Characterize first: RTL-SDR V3 (500 kHz-1766 MHz; direct-sampling 500 kHz-28.8 MHz;
  8-bit; 2.4 MS/s; 1 ppm TCXO; ~$25). Two modes cover conducted + radiated.
- Upgrade if conducted signatures prove discriminating: RSP1A/B (1 kHz-2 GHz, 14-bit
  native <=6.048 MSPS, 10 MHz BW, ~$120) or Airspy HF+ Discovery (500 Hz-31 MHz +
  60-260 MHz, 18-bit DDC, 110 dB BDR, 768 kHz, US$169).
- ADC dynamic range > sample rate for fingerprinting (8/14/18-bit -> 48/84/110 dB).

## 7. SDR host hardware (determined)
The dongle streams I/Q over USB to a host that does FFT/spectrogram and emits
feature vectors. Load is trivial:
- Raw I/Q: V3 4.8 MB/s; RSP1A 12.1 MB/s (max 21.3); Airspy HF+ 3.1 MB/s.
- FFT: 2048-pt @ 50% overlap -> 750-10,410 FFT/s -> a few % of ONE modern core
  (a core sustains ~65k-200k FFT/s).
- RAM: GNU Radio/SoapySDR + Python + NumPy ~ 300-600 MB active; 2 GB comfortable.
- Output: spectrogram feature vectors ~10 kB/s at 10 fps; storage ~MB/day.
- One USB 2.0 port (dongles are USB 2.0), ~5 W continuous.

Tiers:
- Minimum: Raspberry Pi 4 (2 GB) — runs V3/RSP1A at full rate 24/7.
- Comfortable: Pi 4/5 (4-8 GB) or any retired x86 (NUC/laptop).
- The existing zeppo host (i7-8700K, 62 GB) = 10-100x overkill, already running.

Recommendation: use the existing host for the two-evening calibration (fine in
the short term). If signatures warrant a permanent install, add a dedicated
Pi 4/5 near the antenna/panel to (a) escape the big host's PSU RFI near the probe
and (b) decouple 24/7 capture from vLLM/Docker/kernel reboots. Feature vectors then
publish over Ethernet/WiFi into the existing MQTT -> InfluxDB -> Grafana path.

## 8. Two-evening calibration (phase locked in, runs FIRST)
Evening 1 — characterize bands (start radiated, zero risk):
  RTL-SDR V3 + SoapySDR on the host. Loop antenna near the panel.
  For each appliance in a fixed on/off test list: spectrogram 500 kHz-28.8 MHz
  (direct-sampling) and 30 MHz-1 GHz (tuner), diff on-minus-off.
  Log: band, spur locations, magnitude delta, repeatability (3 trials).
  Verdict gate: does radiated carry discriminating signatures for YOUR appliances?
Evening 2 — conducted probe test (only if Evening 1 radiated is insufficient):
  Build/acquire Tier 1 coupler at one outlet. Repeat the same diff, 150 kHz-30 MHz.
  Verdict gate: do conducted signatures discriminate where radiated failed?
Output = a sorted table (appliance x band x discriminability x drift). This table
decides: (a) radiated-only vs conducted needed, (b) receiver upgrade (RSP1A vs
Airspy HF+ vs stay RTL-SDR), (c) permanent install vs bench-only.

Operational notes for the probe set (near-field):
- Keep geometry fixed per appliance (distance + orientation), pick one spot
  (cord vs chassis vs vents) and stick to it; log which spot.
- Use H-field loop for motor/current noise, E-field tip for case/surface noise.
  Log which element was used.
- Diff against the quieter of pre/post "off" captures; allow a settle beat after
  turn-on (transients ring ~1-2 s).

## 9. Open questions (knobs left open)
- Single-phase vs split-phase 240 V (channel budget).
- Utilization-only vs harmonics (tier: IoTaWatt vs ADE9000 build).
- Radiated-only vs conducted (decided by calibration, Section 8).
- Fusion scope: RF as discriminator, arc-fault watchdog, or both.
- Permanent host: existing zeppo box vs dedicated Pi (Section 7).
- AFE board BOM/firmware not yet specified (next after calibration justifies build).
