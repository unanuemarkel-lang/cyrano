# Architecture

## The constraint that shapes everything

A conversational prompter has a latency budget measured against **~200 ms**, the modal gap between speaker turns in human conversation. A naive request/response loop costs 5–9 seconds, and ~4 of those are the irreducible time it takes to *hear* a spoken sentence.

Every architectural decision below exists to work around that.

## Data flow

```
┌───────────────────────────── DEVICE ─────────────────────────────┐
│                                                                  │
│   Air MEMS mic ─────┐                                            │
│   (the room)        │                                            │
│                     ├──► MCU ──► Opus ──► Wi-Fi/BLE ─────────────┼──┐
│   Bone accelerometer┘      │                                     │  │
│   (wearer only)            │      audio as DATA, never A2DP/HFP  │  │
│                            │                                     │  │
│                            ▼                                     │  │
│                     I2S ──► MAX98357A ──► bone transducer        │  │
│                                                                  │  │
└──────────────────────────────────────────────────────────────────┘  │
                                                                      │
                     ┌────────────────────────────────────────────────┘
                     ▼
        ┌────────────────────────────────────┐
        │   Backend — Python / FastAPI       │
        │                                    │
        │   • speaker state machine          │
        │   • rolling conversation context   │
        │   • PRE-GENERATION on their        │
        │     turn-end                       │
        │   • suggestion buffer              │
        │   • per-stage latency logging      │
        └──────────────┬─────────────────────┘
                       ▼
        ┌────────────────────────────────────┐
        │   Qwen3-Omni-30B-A3B-Instruct      │
        │   Novita, OpenAI-compatible        │
        └────────────────────────────────────┘
```

## Why the device is not a Bluetooth headset

A Bluetooth audio peripheral is trapped by profiles: A2DP is high-quality but output-only, HFP is bidirectional but narrowband mono, and any active audio input forces the route to HFP. Recording from one microphone while playing high-quality audio to a Bluetooth sink is an explicitly impossible route on iOS.

Streaming **audio as data** over Wi-Fi or BLE GATT sidesteps this entirely. The OS sees a data peripheral and has no opinion about audio routing. You get genuine full duplex because you own both paths.

Bandwidth is a non-issue: Opus voice is 16–32 kbps, bidirectional needs ~64 kbps, and BLE 5 with the 2M PHY delivers 700 kbps–1 Mbps in practice. The real cost is connection interval (7.5 ms minimum) and its power draw.

## Why there is no phone app

With Wi-Fi, the device talks HTTPS to the backend directly. That deletes from the project: Swift, CoreBluetooth, `AVAudioSession` route and interruption handling, iOS background-execution limits, and App Store review.

The tradeoff is that Wi-Fi only works on known networks and draws considerably more than BLE. **Correct for bench validation, wrong for a shipping wearable.** Add the BLE path and a thin phone client only once Phases 0 and 1 have passed.

## Speaker state machine

Determined by hardware, evaluated in code. No model involved.

```
                     ACCELEROMETER
                  ┌───────────┬───────────┐
                  │  signal   │  silent   │
   ┌──────────────┼───────────┼───────────┤
A  │   signal     │  WEARER   │   OTHER   │
I  ├──────────────┼───────────┼───────────┤
R  │   silent     │   (n/a)   │  silence  │
   └──────────────┴───────────┴───────────┘
```

Policy:

| State | Action |
|---|---|
| **Other speaking** | Transcribe → append context → **pre-generate suggestion into buffer, do not play** |
| **Wearer speaking, normal volume** | Transcribe → append context → **stay silent** |
| **Wearer speaking + wake word** | **Respond** |
| **Double tap** | **Release** the buffered suggestion |
| Silence | Nothing |

Two properties worth noting:

**The wake word is detected only on the bone channel.** The other person can say the wake word all afternoon and nothing fires, because their voice never reaches the accelerometer. This eliminates the entire category of accidental activation, and saves power — the detector only runs while the wearer is speaking.

**Double-tap comes free.** Tap detection is a standard IMU function, using the sensor already present. It is also the most discreet possible trigger: you touch behind your ear.

### Gating rules

- **Mute the accelerometer channel during playback.** The transducer vibrates the same chassis; without this the device detects itself as the wearer speaking.
- **Band-pass the accelerometer to the voice band.** Footsteps, chewing and scratching are structure-borne too. The LSM6DSV16BX audio channel is specified above 1 kHz precisely to sit clear of the motion band.

### An optional second layer

Bone/air ratio: a quiet *voiced* murmur is strong on the accelerometer and weak on the air microphone, which is a usable "this is for the coach" signal without any explicit trigger.

**The trap:** it must be a murmur, not a whisper. A true whisper has no vocal-fold vibration, therefore no bone conduction, therefore the accelerometer does not see it. Counter-intuitive and expensive to discover empirically. Ship the wake word and tap first; treat the ratio as a calibrated layer 2.

## Pre-generation

This is what makes the latency budget work.

```
Other person speaking ─────────────────────────┐
        │                                      │
        │  transcribe, accumulate context      │
        │                                      │
        └─► turn ends ──► generate suggestion ─┘
                               │
                          [ BUFFER ]   ← nothing plays
                               │
        double tap ────────────┴──► play
```

The trigger does not start a request. It **releases audio that already exists**. Time-to-first-syllable becomes playback latency, and the total becomes the ~1.5 s it takes to speak five words.

This mirrors what humans do: formulating a response takes ≥600 ms, yet people answer within 200 ms, because they predict the end of the turn and prepare in advance.

## Output contract

The five-word cap is not a style preference, it is a latency requirement:

> *"You could tell them the Civil War began in 1936 with the July coup"* → **6 seconds**
> *"1936. July coup."* → **1.5 seconds**

The coach supplies ammunition, not scripts. Full prompt specification in [SOFTWARE.md](SOFTWARE.md).

**Silence must be a valid output.** A coach that always speaks is noise. The model needs explicit permission to return nothing, and the system needs to treat that as success rather than failure.

## Model choice

**Qwen3-Omni-30B-A3B-Instruct** — natively end-to-end: audio in, audio and text out in one pass, no separate ASR or TTS in the critical path.

| | |
|---|---|
| First-packet latency | **234 ms** cold-start (reported) |
| Text→voice tool-calling degradation | **1.8 points** — smallest in class (GPT-Realtime-1.5: 4.8) |
| Speech input / output languages | 19 / 10 (Spanish in both) |
| Weights | Apache 2.0 |
| Novita pricing | $0.25/M in, $0.97/M out |

At ~427 audio tokens per minute, an hour of conversation costs roughly **$0.02**. Cost is not a design constraint.

Reasoning models are disqualified regardless of price: GLM-5.2 scores higher on intelligence but has 1.57 s TTFT and is documented as verbose. In voice, verbosity is unskippable time.

Qwen3-Omni **does not do speaker diarization** — which is exactly why the accelerometer exists.

### Hosting

| Option | When |
|---|---|
| **Novita** ($0.25/$0.97 per M) | Default. Audio output confirmed, OpenAI-compatible |
| Alibaba DashScope | Only source of Qwen3.5-Omni and the realtime WebSocket API. Free tier: 1M tokens ≈ 39 audio-hours / 90 days. Endpoints in Singapore and Beijing |
| Fireworks | ~$0.90/M, audio output unconfirmed |
| Self-host | **Not before ~20,000 active voice users.** vLLM-Omni needs a 3-GPU staged deployment (Thinker / Talker / Code2Wav) at $4,100–7,200/month always-warm. At pilot scale that is ~$20 per audio-hour |

OpenRouter does **not** carry any Qwen omni model — only the split ASR and TTS pieces, which is the chained pipeline, not omni.

## Deliberate non-goals

- **Zephyr / nRF5340** — the right production platform, weeks of learning curve. Arduino or ESP-IDF on ESP32-S3 until battery life actually matters.
- **Realtime WebSocket API** — pre-generation plus an explicit trigger removes the need for barge-in, and avoids single-vendor lock-in.
- **Custom PCB** — certified modules only.
- **Always-on cloud recording** — see [RISKS.md](RISKS.md). Personal use only until the consent problem has a real answer.
