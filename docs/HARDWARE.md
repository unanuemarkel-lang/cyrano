# Hardware

All prices verified August 2026. Nothing here requires SMD soldering.

## Bill of materials

### Core

| Part | Role | Price | Source |
|---|---|---|---|
| **XIAO ESP32S3** | MCU. 240 MHz Xtensa dual-core, 8 MB PSRAM (needed for audio buffers), 8 MB flash, Wi-Fi + BLE 5.0, **external rod antenna** | $7.49 | [Seeed](https://www.seeedstudio.com/XIAO-ESP32S3-p-5627.html) |
| **XIAO ESP32S3 Sense** | Same MCU + detachable board with PDM mic, camera, **microSD slot**. Bought only for Phase 0 | $13.90 | [Seeed](https://www.seeedstudio.com/XIAO-ESP32S3-Sense-p-5639.html) |
| **SPH0645LM4H** | Air microphone, I2S, 24-bit, AOP 120 dB SPL, SNR 65 dB. External on leads so it can be placed and decoupled | $6.95 | [Adafruit 3421](https://www.adafruit.com/product/3421) · [Pi Hut](https://thepihut.com/products/adafruit-i2s-mems-microphone-breakout-sph0645lm4h) |
| **MAX98357A** | I2S DAC + class-D amplifier. **1.8 W into 8 Ω at 5 V**, runs 2.7–5.5 V. Screw terminal pre-soldered | $5.95 | [Adafruit 3006](https://www.adafruit.com/product/3006) · [SparkFun](https://www.sparkfun.com/sparkfun-i2s-audio-breakout-max98357a.html) |
| **Bone transducer ×2–3** | 8 Ω, 1 W RMS / 2 W max, 300 Hz–19 kHz, 14 × 21.5 mm, 9.6 g, SPL 90.1 dB 1W/1m | $8.95 ea | see below |
| **2.4 GHz FPC antenna** | 1.16 dBi, IPEX1. Moves the radio away from the skull | $0.50 | [Seeed](https://www.seeedstudio.com/2-4GHz-FPC-Antenna-1-16dBi-for-XIAO-ESP32S3-p-6440.html) |
| **microSD ≤32 GB** | Phase 0 recording. **Must be ≤32 GB, FAT32** | ~€8 | any |
| **USB power bank** | Portable power for Phase 0 without soldering | ~€10 | any |

**Buy 2–3 transducers.** You will destroy one experimenting with gain, and you will want to test multiple skull positions.

> ⚠️ The transducer is frequently out of stock at Adafruit. Verified alternatives:
> [Solectroshop](https://solectroshop.com/en/modulos-arduino/1089-transductor-conduccion-osea-8-ohm-altavoz-1-watt.html) (Spain) ·
> [Opencircuit](https://opencircuit.shop/product/bone-conductor-transducer-wires-8-ohm-1-watt) (Netherlands, EU — no customs) ·
> [DigiKey](https://www.digikey.com/en/products/detail/adafruit-industries-llc/1674/6827067) ·
> [Jameco](https://www.jameco.com/z/1674-Adafruit-Industries-Bone-Conductor-Transducer-with-Wires-8-Ohm-1-Watt_2506295.html)
> [Datasheet](https://www.mouser.com/datasheet/2/737/adafruit_1674_Web-3314756.pdf)

### Own-voice detection

| Part | Role | Price |
|---|---|---|
| **IMU breakout, high ODR** | Prototype own-voice detector. LSM6DS3 reaches 6.6 kHz ODR — enough for voiced-speech fundamentals | ~$10 |
| [**LSM6DSV16BX**](https://www.st.com/en/mems-and-sensors/lsm6dsv16bx.html) | Production part. 6-axis IMU + **audio accelerometer for bone conduction (>1 kHz)** + Qvar, 2.5 × 3.0 × 0.71 mm, 0.95 mA | from $3.95 @1k |

The LSM6DSV16BX is the right chip — it separates the audio channel from the motion/UI channel, which is exactly what you need to reject footsteps and chewing. But it is a **2.5 × 3 mm VFLGA package: not hand-solderable and not breadboardable.** Save it for a custom PCB.

Prototype alternative if you don't want a second IMU: a bone transducer is an electromechanical device and is **reciprocal** — it works as a contact microphone. You need a preamp because the signal is tiny, and quality is mediocre, but for a binary "is the wearer speaking" decision it is sufficient. You are already buying spares.

### Power (Phase 4 only — use USB before that)

| Part | Role | Price |
|---|---|---|
| **LiPo 3.7 V 500 mAh** | JST-PH connector, **protection circuit included** (over-charge, over-discharge, short) | ~$6–8 · [Adafruit 1578](https://www.adafruit.com/product/1578) |
| **470–1000 µF electrolytic** | Bulk reservoir at the amplifier supply. **Not optional** — see below | ~€0.30 |
| PowerBoost 1000C *(optional)* | Charger + load-sharing + 5.2 V boost @1 A | $19.95 · [Adafruit 2465](https://www.adafruit.com/product/2465) |

The XIAO has a **charging IC onboard** — solder the LiPo to the BAT pads and skip the PowerBoost entirely. The MAX98357A runs from 2.7 V so it works directly off the cell, just with less output (~0.4 W into 8 Ω at 3.7 V instead of 1.8 W at 5 V). Try direct first; a boost converter wastes 10–15% in conversion.

### Tools (one-time, ~€85)

Temperature-controlled iron (Pinecil or TS101, ~€40) · 0.6–0.8 mm rosin-core solder (~€8) · breadboard and Dupont jumpers (~€10) · multimeter (~€15) · helping hands and desoldering braid (~€12).

---

## Wiring

The ESP32-S3 I2S peripheral is full-duplex, so the microphone and amplifier **share clocks** and differ only in the data line. One bus for everything.

```
        ┌───────────────────────────────────────┐
        │            XIAO ESP32S3               │
        │                                       │
        │   3V3 ──────┬──────────────────┐      │
        │   5V  ──────┼──────────┐       │      │
        │   GND ──────┼───┬──────┼───────┼──┐   │
        │             │   │      │       │  │   │
        │   I2S SCK  ─┼───┼──┬───┼───────┼──┼─┐ │  shared clock
        │   I2S LRCK ─┼───┼──┼┬──┼───────┼──┼─┼┐│  shared word select
        │   I2S SDIN ◄┼───┼──┼┼┬─┼───────┼──┼─┼┼┼─ data FROM mic
        │   I2S SDOUT─┼───┼──┼┼┼┬┼───────┼──┼─┼┼┼─ data TO amp
        │   GPIO_EN  ─┼───┼──┼┼┼┼┼───────┼──┼─┼┼┼─ amp shutdown
        └─────────────┴───┴──┴┴┴┴┴───────┴──┴─┴┴┴─┘
                          │  ││││           │  │
              ┌───────────┘  ││││           │  │
              │              ││││           │  │
   ┌──────────▼──────────┐   ││││   ┌───────▼──▼────────────┐
   │ SPH0645LM4H         │   ││││   │ MAX98357A             │
   │ 3V GND BCLK LRCL    │   ││││   │ Vin GND BCLK LRC DIN  │
   │            DOUT ────┘   ││││   │                       │
   │ SEL → GND (left ch) │   ││││   │ GAIN → floating (9 dB)│
   └─────────────────────┘   ││││   │ SD   → GPIO           │
                                    │ ┌── C_bulk 470 µF ──┐ │
                                    │ OUT+ ──┐   OUT- ──┐ │ │
                                    └────────┼──────────┼───┘
                                      ┌──────▼──────────▼──────┐
                                      │  Bone transducer 8 Ω   │
                                      └────────────────────────┘
                                                  ║
                                        FIRM PRESSURE against
                                          mastoid or cheekbone
```

### Connection table

| From | To | Note |
|---|---|---|
| 3V3 | mic `3V` | |
| 5V (or LiPo) | amp `Vin` | 5 V gives ~4× the output of 3.7 V |
| GND | mic `GND`, amp `GND` | common ground, mandatory |
| I2S SCK | mic `BCLK` + amp `BCLK` | **shared** |
| I2S LRCK | mic `LRCL` + amp `LRC` | **shared** |
| mic `DOUT` | I2S SDIN | |
| I2S SDOUT | amp `DIN` | |
| mic `SEL` | GND | selects left channel |
| amp `SD` | **GPIO** | see gotcha 1 |
| amp `GAIN` | floating | see gotcha 2 |
| amp `OUT+/OUT−` | transducer | screw terminal, no polarity |

Total soldering: ~6 joints for the mic header, ~7 for the amp header. **The transducer needs none** — the MAX98357A ships with its screw terminal already soldered.

---

## Gotchas that will cost you a weekend each

**1. `SD` goes to a GPIO, not to 3V3.** An always-awake MAX98357A draws 2.4 mA doing nothing — ~58 mAh/day, over 10% of your battery burned in silence. Under MCU control it drops below 1 µA. Two lines of firmware, the highest-return optimisation in the design.

**2. Start at 9 dB gain.** `GAIN` floating = 9 dB, to GND = 12 dB, to Vin = 15 dB. The amp can deliver nearly 2 W into 8 Ω; the transducer is rated 1 W RMS. At 15 dB you cook it.

**3. The bulk capacitor is mandatory.** The class-D amp draws a current spike when it starts speaking. A small LiPo cannot sustain it, the rail sags, and the MCU brown-out resets. You will see "it reboots exactly when it tries to talk" and hunt for it in firmware for days. 470–1000 µF at the amplifier `Vin` fixes it.

**4. Common ground.** The classic breadboard failure: powering the amp from a separate supply and forgetting to tie grounds. Result is noise, or nothing.

**5. Keep transducer leads under 10 cm.** Inductive load plus high-frequency class-D switching equals an antenna equals hum.

**6. Mechanical coupling decides everything.** The number one failure in DIY bone conduction is not electrical. The transducer must be **pressed firmly against bone** — mastoid, behind the ear, or cheekbone. Taped loosely or dangling, you hear nothing and will assume the circuit is broken. Budget real enclosure-design time for clamping pressure. It matters as much as the amplifier.

**7. Decouple the microphone from the transducer.** Same chassis, and the transducer vibrates it by design. Soft-mount the mic in foam, or move it forward toward the hinge — which also aims it better at the person you are talking to.

**8. Keep the antenna off your skull.** Tissue attenuates ~0.4 dB/mm at 2.4 GHz and detunes the antenna by 4–6% depending on body location. The entire ISM band is only 3.4% wide — a 5% shift puts you **outside the band**. Put the antenna on the outward face with the ground plane between it and your head, keep 5–10 mm of standoff, or run the FPC antenna out to a part of the arc that stands off from the skull. This is why sport headsets wrap around the back of the head rather than being two isolated pods.

**9. Never design your own RF PCB.** Use certified modules. Antenna design, matching networks, and radio certification are a specialty. The XIAO boards already carry an FCC ID.

---

## Power budget

> **Corrected 2026-08-08 after external review.** An earlier version quoted
> 12–25 mA for "BLE/Wi-Fi" and concluded 15–20 hours, then compared that to
> Omi. Both were wrong: it conflated two power regimes an order of magnitude
> apart, and Omi runs different silicon.

### Fixed loads

| Block | Draw |
|---|---|
| MEMS microphone | ~1 mA |
| MAX98357A idle | 2.4 mA |
| MAX98357A in shutdown | **< 1 µA** |
| MAX98357A playing | 60–250 mA peak |

A coach speaking 3 seconds every two minutes is a **~2.5% duty cycle**, which turns those output peaks into 2–3 mA average. **The output path is not the problem.**

### The radio is the problem

| Configuration | MCU average | Total | 500 mAh cell |
|---|---|---|---|
| ESP32-S3, **Wi-Fi** streaming continuously | **~120 mA** (peaks 300+) | ~125 mA | **~4 hours** |
| ESP32-S3, active, radio idle | ~23 mA | ~28 mA | ~18 hours |
| ESP32-S3, **BLE** streaming Opus | ~30–50 mA *(estimate — measure it)* | ~35–55 mA | 9–14 hours |
| nRF5340, BLE + Opus | 12–20 mA | ~15–23 mA | 24 h+ |

**Wi-Fi is a bench transport, not a wearable one.** Four hours is enough for a day of desk testing. It is not enough to wear.

**BLE on ESP32-S3 is a workable middle**, but it is not nRF-class and the figure above is an estimate, not a measurement. Verify it before designing an enclosure around a battery size.

**The 24-hour class requires leaving ESP32-S3.** Omi reports 24 h+ of continuous capture on an nRF5340 with BLE and Opus — different chip, different power architecture, not reachable from ESP32-S3 by tuning firmware. Treat the migration to nRF5340 + Zephyr as the price of all-day battery, and pay it only once Phases 1 and 0 have passed.

## Safety

You are wearing a lithium cell against your skull. Buy cells **with protection circuitry** (Adafruit, Pimoroni and SparkFun cells include it; loose AliExpress cells often do not), design a rigid cavity so the cell cannot be bent or punctured, and do not charge unattended. A punctured LiPo burns.
