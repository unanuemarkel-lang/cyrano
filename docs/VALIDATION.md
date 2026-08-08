# Validation plan

## Principle

Five unknowns. **The two that can kill the project require building nothing.** They are independent of each other, so run them in parallel. Phases 2–4 are only worth starting if both pass.

Write every kill criterion down **before** collecting data.

| Phase | Question | Cost | Time |
|---|---|---|---|
| **0** | Does a body-worn mic capture the other person in real noise? | $14 + card | a weekend |
| **1** | Is a 3-word prompt at 2 s useful or distracting? | €0 | a weekend |
| 2 | Is bone-conducted speech intelligible at sane power? | $25 | a weekend |
| 3 | Does own-voice detection hold up? | $10 | a weekend |
| 4 | Integration, battery, enclosure | rest | weeks |

---

## Phase 0 — capture

**The single most likely way this project dies.** In an 80–85 dB bar, a speaker at 60 dB sits 20–25 dB below the noise floor. No gain recovers information that was never captured.

### Setup

XIAO ESP32S3 Sense + microSD (≤32 GB, FAT32) + USB power bank. **Zero soldering** — the Sense board carries the PDM microphone and card slot internally over the B2B connector.

Flash Seeed's WAV recorder, 16 kHz mono, PSRAM enabled.

### Protocol

Clip it to your lapel and record ten minutes of real conversation in each of:

1. **Quiet room** — the control. If this fails, something is wrong with the setup.
2. **Street** — traffic, wind, movement.
3. **Busy bar** — the realistic worst case and the one that matters.

Vary distance to the other speaker: 0.5 m, 1 m, across a table.

### Analysis

Run the WAVs through the model and **read the transcripts. Do not judge by listening** — your ear reconstructs what the model cannot.

Measure:

- Word error rate on **the other speaker** specifically, not the overall transcript. Your own voice will always come through; that is not the question.
- The distance at which it breaks.
- Degradation from quiet → street → bar.

### Kill criterion

> If the other speaker's transcript is unusable in the bar condition, better firmware and better models do not fix it. Change the microphone strategy, change placement, or change the use case.

An outward-facing directional mic, a mic array, or a chest/lapel position rather than the head are all live options — but you need this data before choosing.

---

## Phase 1 — the coach

Runs entirely on a laptop with wired earbuds. No hardware, no soldering, no dependency on Phase 0.

### Setup

Python loop implementing the real design: rolling context, pre-generation on the other speaker's turn-end, buffered suggestion, manual key as a stand-in for double-tap, 5-word output cap.

Point at Novita. DashScope's free tier (1M tokens ≈ 39 audio-hours over 90 days) also covers this at zero cost if you want a second model for comparison.

### Protocol

Use it yourself in **five real conversations**. Not demos, not tests with a friend who knows what you are doing. Real ones where you actually want to say something well.

### Measure

| Metric | Why |
|---|---|
| Times you triggered it per conversation | Engagement |
| Times the suggestion was actually useful | Precision |
| Times it fired and you ignored it | Noise rate |
| Perceived latency: trigger → first syllable | Must stay under ~800 ms |
| How often the model correctly returned silence | Is the restraint working |
| Whether you kept reaching for it by conversation five | The only metric that matters |

### Kill criterion

> If it distracts more than it helps after five real conversations, stop. No hardware fixes this.

This is the honest one. The instinct will be to blame the model, the prompt, the latency. Sometimes that is true — but if a 3-word prompt at 2 seconds is fundamentally the wrong interaction for live conversation, the entire product is wrong, and $80 of components will not reveal that any better than a laptop does.

---

## Phase 2 — bone conduction output

Breadboard, USB powered, no battery.

```
XIAO ESP32S3 ──I2S──► MAX98357A ──► bone transducer
                          GAIN floating (9 dB)
                          SD on GPIO
                          470 µF at Vin
```

Thirteen solder joints, all header pins. The transducer goes into the pre-soldered screw terminal.

### Measure

- **Intelligibility** of synthesised speech, quiet and in noise.
- **Skull position**: mastoid vs. cheekbone vs. above the ear. Expect large differences.
- **Clamping pressure required.** This is the number one DIY failure mode — loose mounting produces no sound and you will blame the electronics.
- **Actual power needed.** Start at 9 dB and only increase if genuinely too quiet. At 15 dB you destroy the transducer.
- Whether the 300 Hz low-frequency cutoff hurts intelligibility. For speech it should not.

---

## Phase 3 — own-voice detection

Add a high-ODR IMU breakout (LSM6DS3 reaches 6.6 kHz) or use a spare transducer as a contact mic with a preamp.

### Validate the truth table

| Accelerometer | Air mic | Expected |
|---|---|---|
| signal | signal | wearer |
| silent | signal | other |
| silent | silent | silence |

### Specifically test the failure modes

- **Playback feedback.** Confirm that gating the accelerometer channel during playback prevents the device detecting itself.
- **Motion artifacts.** Walking, chewing, scratching the housing. Confirm the voice-band filter rejects them.
- **Wake word on the bone channel only.** Have someone else say the wake word repeatedly. It must never fire.
- **Double-tap** false positive rate while walking.

---

## Phase 4 — integration

LiPo on the XIAO BAT pads, FPC antenna standing off from the skull, Wi-Fi to the backend, printed enclosure with real clamping pressure.

Measure actual battery life against the ~15–20 h estimate. Add BLE and a phone client only when you need to leave known Wi-Fi.

---

## What "success" looks like

There is no version of this where the answer is obvious. The realistic good outcome after Phases 0 and 1:

- The other speaker is transcribable in moderate noise but marginal in a loud bar
- You reach for the coach in perhaps one conversation in three, and when you do it is genuinely useful
- Latency feels acceptable with pre-generation and unacceptable without

That is enough to justify Phase 2. Anything less is not, and the discipline is to say so.
