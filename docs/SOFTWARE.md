# Software

## Stack

| Layer | Choice | Why |
|---|---|---|
| Intelligence | Qwen3-Omni-30B-A3B-Instruct on Novita | End-to-end audio, 234 ms first packet, Apache 2.0 weights |
| Backend | Python + FastAPI, OpenAI SDK pointed at Novita | Provider-swappable, instrumentation for free, API key off-device |
| Transport | Wi-Fi / HTTPS direct from device | No phone client |
| Firmware | Arduino or ESP-IDF on ESP32-S3 | **Not Zephyr** — see [ARCHITECTURE.md](ARCHITECTURE.md) non-goals |
| Capture | I2S, 16 kHz mono, 24-bit, **AGC off** | 16 kHz is the model's native rate |
| Diarization | Accelerometer state machine, in code | The model does not do it |

## Backend

```
FastAPI
  └─ POST /coach/turn
       ├─ Novita  base_url = https://api.novita.ai/openai
       │          model    = qwen/qwen3-omni-30b-a3b-instruct
       │          modalities = ["text", "audio"]
       │          audio      = { voice: "chelsie", format: "wav" }
       ├─ rolling conversation context (who said what)
       ├─ suggestion buffer
       └─ log: transcript, speaker, per-stage latency
```

Audio goes in the OpenAI content-array format: base64-encoded WAV alongside the text prompt.

### Why route through your own backend

Calling the provider directly from the device is one hop faster (~50–150 ms) but you give up: the API key staying server-side, instrumentation, the ability to swap Novita → DashScope → self-host without touching firmware, and the seam where memory, tools and retrieval eventually plug in.

Take the hop.

### Instrumentation — non-negotiable

The prototype exists to produce measurements. Log every turn:

| Field | Why |
|---|---|
| Transcript, per speaker | The only honest measure of capture quality |
| Speaker label (from accelerometer) | Validates the state machine |
| Latency per stage: capture → upload → inference → playback | When it feels slow you need to know which of the four |
| Suggestion text and whether it was released | Distinguishes "generated garbage" from "generated fine, never used" |
| Environment tag (quiet / street / bar) | The SNR variable |
| Comprehension failures: wearer repeats or rephrases within 10 s | Separates a model problem from a demand problem |

That last one decides what a failed pilot means:

| Observation | Diagnosis | Action |
|---|---|---|
| Understood fine, stopped using it | Demand problem | Kill |
| Kept using it, tired of repeating | Model / capture problem | Fix capture, retry |

Write the threshold down **before** looking at data: if comprehension failure exceeds **10%**, the run is inconclusive — re-run the noisy condition on a better model rather than concluding anything about demand.

## Prompt specification

The output contract is a latency requirement, not a style preference.

```
You are a conversation prompter. The wearer is mid-conversation and cannot wait.

MUST: reply in 5 words or fewer.
MUST: supply ammunition, never a script — a fact, a name, a date, an angle.
MUST: reply in the language of the conversation.
MUST: return nothing when you have nothing useful. Silence is a valid answer
      and is preferred over filler.

NEVER: produce a full sentence for the wearer to repeat.
NEVER: explain, hedge, or preface.
NEVER: address the other party.
```

Examples of the shape:

| Context | Bad (6 s) | Good (1.5 s) |
|---|---|---|
| Spanish Civil War | "You could mention the war started in 1936 with the July military coup" | "1936. July coup." |
| Salary negotiation | "It might be worth asking what the band for this role is" | "Ask the band." |
| Nothing useful | "I'm not sure what would help here" | *(silence)* |

Silence being a first-class output is the difference between a coach and a nuisance. Instrument how often it fires — if it never does, the prompt is not working.

## Firmware

### Phase 0 — recorder

Seeed's official example does this already:

- [Microphone Usage for Sense](https://wiki.seeedstudio.com/xiao_esp32s3_sense_mic/) — mic to SD card
- [Keyword Spotting](https://wiki.seeedstudio.com/xiao_esp32s3_keyword_spotting/) — includes the full WAV recorder
- [Getting Started](https://wiki.seeedstudio.com/xiao_esp32s3_getting_started/)
- [makerguides tutorial](https://www.makerguides.com/record-audio-with-xiao-esp32-s3-sense/)
- [Example repo](https://github.com/Mjrovai/XIAO-ESP32S3-Sense)

Set 16 kHz mono. **Enable PSRAM in the Arduino IDE before uploading** — the recorder does not start otherwise, and this is the most common failure with this board. microSD must be ≤32 GB, FAT32.

### Later phases

Full-duplex I2S: shared SCK and LRCK, `SDIN` from the microphone, `SDOUT` to the amplifier. Amplifier `SD` on a GPIO so it can drop to sub-µA between utterances.

Device-side loop:

```
capture 16 kHz mono, flat, no AGC
  │
  ├─ air VAD ────────┐
  ├─ accel VAD ──────┤──► speaker state ──► label the chunk
  │                  │
  └─ Opus encode ────┴──► HTTPS POST to backend

on double-tap interrupt ──► request buffered suggestion
on playback start ───────► mute accel channel
on playback end ─────────► unmute, amp SD low
```

### Speaker-aware gain

Because the accelerometer tells you who is talking, apply gain per speaker in software — attenuate when it reports the wearer, boost when it reports the other party.

Do **not** use AGC. It pumps: your own voice ducks theirs every time you speak, which is precisely the failure the flat 24-bit capture avoids.

## Cost model

At ~427 audio tokens per minute of audio, on Novita at $0.25/M in and $0.97/M out:

| Usage | Cost |
|---|---|
| 1 hour of conversation | ~$0.02 |
| Daily user, 20 min/day | **~$0.19/month** |
| 20-person pilot, two weeks | **< $2 total** |

Inference cost is roughly 1% of any plausible subscription. It is not a design variable — do not optimise it, and do not let it justify self-hosting.
