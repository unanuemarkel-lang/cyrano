# Cyrano

**An open-source bone-conduction wearable that listens to your conversation and whispers you the next thing to say.**

Not a note-taker. Not a meeting recorder. A live conversational prompter — three words in your ear, while the conversation is still happening.

> **Status: specification only. Nothing has been built.**
> This repository is a complete, honest design document published so someone can pick it up. Every number here is sourced. The parts that will most likely kill this project are documented as prominently as the parts that might work.

---

## The idea

You are in a conversation. The other person says something you could answer better if you had one fact, one name, one date, one angle. A device on your head heard it too, and 1.5 seconds later it says *"1936. July coup."* into your skull, audibly to you and nobody else.

That is the entire product. Everything below is what it takes to make that sentence true, and the evidence about whether it can be.

## Why bone conduction

Not aesthetics. Three functional reasons:

1. **Open ear.** You cannot occlude your hearing during a conversation. Any in-ear device is disqualified.
2. **Private output.** Bone conduction is audible to the wearer and essentially silent to the person across the table.
3. **A second sensor for free.** Skull contact enables an accelerometer-based own-voice detector, which turns out to be the key to the hardest software problem in this product (see [Addressee detection](#addressee-detection)).

## Why not just use Shokz / AirPods / any existing headset

Two hard blockers, both verified:

**The microphone is engineered to delete the other person.** Sport headsets beamform on the wearer's mouth and aggressively reject everything else — the Shokz OpenRun Pro 2 advertises filtering 96.5% of background noise. That "background noise" is the exact signal this product needs.

**Bluetooth audio profiles make it impossible anyway.** A Bluetooth headset cannot do high-quality output and microphone input simultaneously — A2DP is output-only, HFP is bidirectional but narrowband mono. Per Apple's documentation, *any* active audio input forces the route to HFP; recording from the phone's built-in microphone while playing to an A2DP headset is an explicitly impossible route on iOS.

Building the device sidesteps both. Custom hardware streams audio as **data** over Wi-Fi or BLE — it is not a Bluetooth audio peripheral, so the operating system never applies a profile to it, and the microphone is one you chose and placed yourself.

---

## The four non-obvious findings

These cost the most to work out and are the most useful part of this document.

### 1. Four seconds of the latency are irreducible

The naive loop — wait for them to finish, transcribe, generate, speak — measures out like this:

| Stage | ms |
|---|---|
| Voice-activity detection on end-of-turn | 300–700 |
| Upload to backend | 50–150 |
| Backend → inference provider | 50–200 |
| Model first audio packet | 600–1500 |
| Playback buffer | 100–200 |
| Bluetooth output latency | 50–150 |
| **To first syllable** | **1,200–2,900** |
| **Listening to a 10–15 word suggestion** | **4,000–6,000** |
| **Total** | **5–9 seconds** |

The modal gap between turns in human conversation is **~200 ms**, consistent across languages. That is 25–45× over budget.

And with a zero-latency model you would *still* need ~4 seconds to hear a spoken sentence. This is not an inference problem. It is the physics of speech.

**The fix is not speed, it is prediction and brevity:**

- **Pre-generate.** Humans don't react to end-of-turn, they predict it and prepare in advance. Generate the suggestion *while* the other person is still talking, buffer it, and release it on a trigger. This collapses the first 1.2–2.9 s to near zero.
- **Telegraphic output, hard capped at 5 words.** Not *"You could tell them the Civil War began in 1936 with the July coup"* (6 seconds) but *"1936. July coup."* (1.5 seconds). The coach supplies ammunition, not scripts.

Combined: **under 2 seconds**, which a natural filler ("right… well…") can cover.

### 2. Two separate problems: who is speaking, and who they are speaking to

These get conflated constantly — an earlier version of this document conflated
them too. They have different solutions and different reliability.

**Who is speaking — solved in hardware.**

Qwen3-Omni does not do speaker diarization; it returns flat text for a two-voice
recording. So use two sensors with different physics:

| Sensor | Hears |
|---|---|
| Air MEMS microphone | The room: the other person **and** you |
| Bone-conduction accelerometer | **Only you**, via skull vibration |

An accelerometer is a mechanical sensor with essentially no acoustic
sensitivity. Your voice reaches it structurally (vocal folds → skull → chassis →
proof mass). The other person's voice arrives as airborne pressure and cannot
use that path. This is why the technique is used for own-voice detection in jet
cockpits and factories — the isolation is a property of the transducer, not a
threshold.

|  | Accelerometer: signal | Accelerometer: silent |
|---|---|---|
| **Air mic: signal** | **wearer speaking** | **other person speaking** |
| **Air mic: silent** | (n/a) | silence |

This is robust because it is physical, but it is **not free of false positives**.
Chewing, footsteps, scratching the housing and the device's own transducer are
all structure-borne and all reach the accelerometer. They are rejected by
band-passing to the voice range and by muting the channel during playback — both
verifiable, neither automatic. Phase 3 exists to measure the residual rate.

**Who they are speaking to — solved in UX, not physics.**

Nothing in the table above tells you whether the wearer is addressing the coach
or the person in front of them. That distinction is resolved by an **explicit
trigger**: a double tap on the housing, or a wake word.

This is a deliberate design decision, not a physical guarantee, and it is worth
being honest about the difference. What the accelerometer *does* buy is that the
trigger becomes reliable: run wake-word detection **only on the bone channel**
and the other person can say the wake word all afternoon without firing
anything, because their voice never reaches that sensor.

Implicit addressing — inferring intent from a quiet murmur, or from a classifier
over the transcript — is a later layer, and a risky one. A false positive means
the coach interrupts you mid-sentence in front of the person you are talking to.

### 3. Saturation is not the risk; SNR is

A natural worry: if the microphone is on your temple and sensitive enough to hear someone a metre away, won't your own voice clip it?

No. The numbers:

| | dB SPL |
|---|---|
| SPH0645 acoustic overload point | **120** |
| Your own voice at your own ear | ~75–85 |
| Other person at 1 m | 60–68 |
| Microphone noise floor | ~29 |

The gap between your voice and theirs is **10–15 dB, not 40**, and your voice sits 35–45 dB below overload. More fundamentally, a digital MEMS microphone has **no gain control to get wrong** — it is a fixed-sensitivity 24-bit I2S device covering the whole range. You capture flat and scale in software. The "set it for them and blow out on yourself" failure mode is analog-microphone intuition and does not apply.

**The real risk is signal-to-noise.** In an 80–85 dB bar, the other person at 60 dB is 20–25 dB *below* the noise floor. No gain recovers information that was never captured. This is the single most likely way the project dies, and it is what [Phase 0](docs/VALIDATION.md) exists to measure.

Two consequences: **disable AGC** (it will duck the other person every time you speak — the failure you intuited, caused by software), and use the accelerometer to apply **speaker-aware gain**, since you know who is talking.

### 4. Your own output is your worst noise source

The bone transducer vibrates the chassis by design. That vibration reaches both the accelerometer (which will report "the wearer is speaking") and the air microphone package. Gate the accelerometer channel during playback — you know exactly when you are playing — and mechanically decouple the microphone from the transducer with soft mounting.

---

## Architecture

```
┌──────────────────────── DEVICE ────────────────────────┐
│                                                        │
│  Air mic (I2S MEMS) ──┐                                │
│                       ├─► MCU ──► Wi-Fi / BLE ─────────┼──┐
│  Bone accelerometer ──┘   │       (audio as DATA,      │  │
│                           │        never A2DP/HFP)     │  │
│                           ▼                            │  │
│                    I2S ──► Class-D amp ──► bone        │  │
│                                            transducer  │  │
│                                                ║       │  │
│  LiPo + charger                        pressed to      │  │
│                                        mastoid/cheek   │  │
└────────────────────────────────────────────────────────┘  │
                                                            │
                          ┌─────────────────────────────────┘
                          ▼
                  Backend (Python / FastAPI)
                    • rolling conversation context
                    • pre-generation on their turn-end
                    • suggestion buffer
                    • per-stage latency logging
                          │
                          ▼
                  Qwen3-Omni-30B-A3B-Instruct
                  (Novita, OpenAI-compatible)
```

**No phone app.** With Wi-Fi the device talks HTTPS to the backend directly. That removes Swift, CoreBluetooth, `AVAudioSession`, App Store review, and iOS background-execution limits from the project entirely. Add BLE and a phone app only when you need to leave known networks.

Full detail: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)

## Bill of materials

Roughly **$80 in parts** plus ~€85 of one-time tools. Thirteen solder joints, all 2.54 mm through-hole — no SMD.

| Part | Role | Price |
|---|---|---|
| XIAO ESP32S3 | MCU, Wi-Fi + BLE, 8 MB PSRAM | $7.49 |
| XIAO ESP32S3 Sense | Onboard mic + microSD, for Phase 0 only | $13.90 |
| SPH0645LM4H | Air microphone, I2S | $6.95 |
| MAX98357A | I2S DAC + class-D amp, 1.8 W into 8 Ω | $5.95 |
| Bone transducer ×2 | 8 Ω, 1 W, 300 Hz–19 kHz, 9.6 g | $8.95 ea |
| IMU breakout (high ODR) | Own-voice detection | ~$10 |
| 2.4 GHz FPC antenna | Keeps the radio off your skull | $0.50 |
| LiPo 500 mAh + cap | Power (Phase 4 only) | ~$8 |

Full table with links, sourcing, and wiring: [docs/HARDWARE.md](docs/HARDWARE.md)

## Validation plan

Five unknowns. **The two that can kill the project require building nothing — start with those, Phase 1 first.**

| Order | Phase | Question | Cost | Kill criterion |
|---|---|---|---|---|
| **1st** | **1** | Is a 3-word prompt at 2 s useful, or does it make you converse worse? | **€0** | You stop reaching for it by the fifth real conversation |
| **2nd** | **0** | Does a body-worn mic capture the other person in a bar? | $14 | Transcript of the other speaker unusable in noise |
| 3rd | 2 | Is bone-conducted speech intelligible at sane power? | $25 | — |
| 4th | 3 | Does own-voice detection hold up? | $10 | — |
| 5th | 4 | Integration, battery, enclosure | rest | — |

Phase 1 comes first because it costs nothing and tests the question that decides the product. It must be measured **against a baseline of uncoached conversations**, not against how it felt — see risk 4. Phases 2–4 are only worth starting if 1 and 0 both pass.

Detail and protocols: [docs/VALIDATION.md](docs/VALIDATION.md)

## Honest risk register

Summarised; full version with sources in [docs/RISKS.md](docs/RISKS.md).

- **Legal.** Recording a conversation you participate in is legal in Spain and covered by GDPR's household exemption for strictly personal use. **That exemption ends the moment this has users.** ePrivacy Art. 5(1) requires consent from *all* parties; GDPR Chapter V additionally makes sending that third party's voice to a non-EEA inference provider a cross-border transfer. **When Meta acquired Limitless it withdrew the service from the EU and UK entirely** rather than operate it there — the best-resourced possible actor concluded exit was cheaper than compliance.
- **Cognitive.** Listening to a prompt competes with listening to a person. The irrelevant speech effect is well documented, and self-report will not detect it: feeling equipped and conversing worse coexist perfectly well.
- **Market.** Amazon acquired Bee (Jul 2025). Meta acquired Limitless (Dec 2025). Humane raised $230M, shipped <10,000 units, sold to HP for $116M. Nobody has built a standalone business here. OpenAI's screen-free, behind-the-ear "Sweetpea" is in prototyping.
- **Technical.** SNR in noisy rooms is the top risk. Then pre-generation firing on incomplete context, mechanical feedback, and antenna detuning against the skull (tissue shifts resonance 4–6%; the ISM band is only 3.4% wide, so a 5% shift puts you outside it entirely).
- **Battery.** ~4 hours over Wi-Fi, not the 15–20 h an earlier version of this document claimed. All-day operation requires moving from ESP32-S3 to nRF5340.

## Prior art

| Project | What it is | Why it matters here |
|---|---|---|
| [Omi](https://github.com/BasedHardware/omi) | MIT open-source AI pendant, nRF5340 + Zephyr, 24 h capture | Closest existing thing. Full schematics, PCB, BOM. **Capture-only — no audio output.** That gap is this project. |
| Bee / Limitless | Always-on AI pendants | Both acquired within 18 months. Neither built a standalone business. |
| iFLYTEK translation earbuds | Bone + air conduction, LLM translation, $299 | Proves the two-sensor approach commercially. |
| Humane AI Pin | $700 screenless assistant | The canonical failure. Read the post-mortems first. |

## Getting involved

Nothing is built. **The author is running Phases 1 and 0 and will publish the
results here, positive or negative.** An earlier version of this section asked
contributors to run Phase 0 instead — which was asking strangers to do the €0
and $14 experiments while keeping the writing. That was the wrong way round.

Useful contributions in the meantime:

- **Tell us where this is wrong.** Every correction in the revision history came
  from someone reading the spec critically. That is worth more than agreement.
- **Independent measurements**, if you happen to have the parts already — a
  second data point on capture in noise, or on ESP32-S3 BLE current draw, which
  is currently an estimate rather than a measurement.
- **Prior art we missed**, especially anything that already tried live
  conversational prompting and failed. Negative results are the scarce resource
  in this category.

If you run anything, open an issue with the raw data, including failures. A
well-documented failure saves everyone else the $80.

## Revision history

**2026-08-08** — Second pass after external review. Four corrections to things
the first version got wrong: an overstated battery budget (3–4×), conflating
speaker diarization with addressee detection, pre-generation described without
acknowledging it fires on incomplete context, and a legal section that missed
GDPR Chapter V transfers and Meta's EU withdrawal of Limitless. One new risk
added (cognitive interference). Validation order swapped. Details in
[docs/RISKS.md](docs/RISKS.md#revision-history).

## License

MIT. Hardware designs, firmware, and documentation.
