# Risk register

Ordered by how likely each is to end the project. Sources at the bottom.

---

## 1. Legal — the consent problem 🔴

**This is the constraint that decides whether Cyrano can ever be a product rather than a personal tool.**

For strictly personal use by one person:

- In Spain, recording a conversation **you participate in** is legal — Art. 197 of the Criminal Code protects conversations you are *not* party to.
- GDPR's household exemption (Art. 2(2)(c)) covers purely personal and domestic activity.

Both of those protections **evaporate the moment this has users**:

- **ePrivacy Art. 5(1)** prohibits listening, recording, storage or surveillance of communications without the consent of *the users concerned* — plural. In a conversation, that includes the person across from you, who consented to nothing.
- Processing locally instead of in the cloud improves the data-transfer story significantly, but **does not solve this**. Art. 5(1) governs the recording, not where the inference runs.
- **GDPR Chapter V — international transfer.** *Added after review; the first version missed this.* The other person's voice does not merely get recorded, it **leaves the EU**. Novita, Fireworks and DashScope all serve from outside the EEA — DashScope's realtime endpoints are Singapore and Beijing. That is a cross-border transfer of biometric-adjacent personal data belonging to someone who never consented, and it is a separate violation from the recording itself. Self-hosting inside the EU fixes the transfer. It does not fix the consent.
- **EU AI Act Art. 43** requires the AI component's conformity assessment to be integrated into the Radio Equipment Directive assessment. Your radio certification becomes contingent on your AI compliance.
- **GPSR** has applied to consumer products including AI-enabled ones since December 2024.

### The precedent nobody should ignore

When Meta acquired Limitless in December 2025, it **discontinued the service in
the EU, UK, Brazil, Israel, South Korea, Turkey and China**. Users had until
19 December 2025 to export their data before permanent deletion.

Read that carefully. Meta — with the best privacy counsel money can buy and an
existing EU regulatory apparatus — concluded it was cheaper to **exit the
European market entirely** than to operate an always-listening conversation
recorder inside it.

If you are building this in Spain, that is not a footnote. It is the most
informative data point in this entire document.

**Implication:** ship this as personal hardware, open source, build-it-yourself. Any commercial path needs a real answer to third-party consent — a visible recording indicator, a consent gesture, on-device-only processing with no retention and no transfer, or a use case where all parties are participants who opted in. Solve that before writing a business plan, not after.

---

## 2. Signal-to-noise in real rooms 🔴

The top *technical* risk, and the reason Phase 0 exists.

In an 80–85 dB bar, a speaker at 60 dB sits **20–25 dB below the noise floor**. No amount of gain recovers information that was never captured, and the gap between your own voice and theirs is only 10–15 dB, so you cannot separate them by level alone.

Note what this risk is *not*: saturation. The SPH0645 has a 120 dB SPL acoustic overload point and your own voice reaches ~75–85 dB at your ear — 35–45 dB of headroom. A digital MEMS microphone has no gain control to set incorrectly.

**Mitigations, in order of cost:** lapel or chest placement instead of head; outward-facing directional mic; two-mic beamforming; ultimately accepting that loud bars are out of scope.

---

## 3. Demand 🔴

Nobody has demonstrated that people want a machine talking in their ear during conversation for more than a novelty week.

The category's track record:

| | Outcome |
|---|---|
| **Humane AI Pin** | $230M raised, elite team, **<10,000 units shipped**, sold to HP for $116M |
| **Bee** | Acquired by Amazon, July 2025 |
| **Limitless** | Acquired by Meta, December 2025; Pendant no longer sold, and **service withdrawn entirely from the EU and UK** (see risk 1) |

Read that as two things at once. There is acquisition appetite — but **none of the three built a standalone business**, and all three had working products. They did not die from token costs or latency. They died because people did not come back the next day.

Phase 1 is the cheapest possible test of this, and it costs nothing.

---

## 4. Cognitive interference — the risk this document originally missed 🔴

*Added 2026-08-08 after external review.*

Listening to a three-word prompt is not free. Speech competes with speech: the
**irrelevant speech effect** is a well-documented impairment of verbal working
memory and comprehension when speech-like sound is present during a verbal task.
A conversation is a verbal task. So is parsing a prompt.

Two honest qualifications:

- The design mitigates this more than the raw effect suggests. Playback is
  released by the wearer's own double tap, so it lands in a gap they chose,
  not on top of the other person's speech. It is closer to a stage prompter
  than to a distractor track.
- But the mitigation is untested, and the wearer may tap badly, or the other
  person may resume talking.

**The methodological trap is worse than the effect.** The feeling of having a
superpower and the fact of conversing worse coexist perfectly well. Self-report
will not detect this — a wearer who enjoys the device will rate it useful
regardless of whether their conversations got better.

**Phase 1 must therefore measure performance, not sensation.** Record the
conversations. Score them afterwards, or have the other party score them, blind
to whether the coach was active. If that is too heavy, at minimum alternate
coached and uncoached conversations and compare, rather than asking yourself how
it felt.

---

## 5. Latency floor 🟠

~4 seconds of the naive 5–9 second loop is the irreducible time to *hear* a spoken sentence. A zero-latency model does not fix it.

Mitigated by pre-generation plus a hard 5-word output cap, which brings it under 2 seconds. But this means the interaction is permanently constrained: **the coach can never explain anything.** If the useful version of this product turns out to require sentences, the product does not exist in audio form. Reading is 3–5× faster than listening; a watch glance is 0.5 s. Worth testing as an alternative modality even though it is socially more visible.

---

## 6. Mechanical feedback 🟠

The bone transducer vibrates the chassis by design. That energy reaches the accelerometer (which will report the wearer speaking) and the air microphone package.

Mitigated by gating the accelerometer during playback — you know exactly when you are playing — plus soft-mounting the microphone and physically separating it from the transducer. Straightforward, but it must be designed in rather than discovered.

---

## 7. Antenna detuning against the skull 🟠

Tissue attenuates ~0.4 dB/mm at 2.4 GHz and shifts antenna resonance **4–6%** depending on body location. The entire ISM band is **3.4% wide** — a 5% shift puts you outside it, not merely degraded.

Geometry makes it worse: device on the mastoid, phone in a trouser pocket, head and torso in between.

Mitigated by outward-facing antenna with the ground plane between it and the skull, 5–10 mm standoff, or an FPC antenna routed to a part of the arc that stands away from the head. Choose a board with an external antenna connector (XIAO ESP32S3 has IPEX1; XIAO nRF52840's ceramic antenna is fixed and offers no escape).

---

## 8. Competition and timing 🟠

The category is not empty:

- **OpenAI "Sweetpea"** — screen-free, behind-the-ear, in prototyping since November 2025, reveal expected late 2026 (some reports suggest a smart speaker ships first, in 2027). Same form factor, $6.5B of acquired design capability behind it.
- **Apple** — Live Translation on AirPods Pro 3, **retrofitted to AirPods Pro 2 and AirPods 4**. The flagship feature of an AI audio wearable is now free on hardware people already own.
- **Meta** — 7M+ Ray-Ban units sold in 2025, 69% smart-glasses share in Q1 2026. The category's money and attention are in glasses, not audio-only.
- **iFLYTEK** — bone + air conduction earbuds with LLM translation, $299, shipping.
- Generic "ChatGPT-powered bone conduction headphones" are already on Alibaba and Amazon.

Every efficiency gain in open models lowers the floor for **everyone simultaneously**. Cheap intelligence dissolves moats rather than building them. This is a reason to build it for yourself and publish it openly, not a reason to build a company.

---

## 9. Build effort 🟡

Components are under $80. The real cost is time.

Mitigated by choosing ESP32-S3 with Arduino/ESP-IDF over nRF5340 with Zephyr — hardware people consistently underestimate the RTOS learning curve, and the ESP32 audio path is far better documented. Migrate to nRF5340 only when multi-day battery life genuinely matters.

Never design a custom RF PCB. Certified modules only.

---

## 10. Battery 🟡

Always-on listening is brutal on wearables: Amazon's Bee drops from an advertised 7 days to 1.5–2 days in real use, a >70% reduction.

The estimate here (~20–30 mA average, 15–20 h on 500 mAh) depends on a low playback duty cycle and on the amplifier shutdown pin being under MCU control. If the coach ends up talking far more than 2.5% of the time, redo the budget.

---

## Sources

Latency and turn-taking: modal inter-turn gap ~200 ms across languages; response formulation ≥600 ms with predictive preparation. Microphone specs: SPH0645LM4H, AOP 120 dB SPL, SNR 65 dB, 24-bit I2S. Own-voice SPL at ear ~75 dB+; conversational speech 60–68 dB at 1 m. Model: Qwen3-Omni technical report (arXiv 2509.17765), 234 ms first-packet cold start; vLLM-Omni serving benchmarks (632 ms audio TTFP, RTF 0.47 at concurrency 64 on a 3-GPU staged deployment). Bluetooth: Apple AVAudioSession documentation on A2DP duplex routing. RF: 2.4 GHz tissue attenuation and wearable antenna detuning literature. Legal: ePrivacy Directive Art. 5(1), GDPR Art. 2(2)(c) and Chapter V, EU AI Act Art. 43, GPSR, Spanish Criminal Code Art. 197. Market: reported acquisitions and shipment figures as of August 2026; Meta's withdrawal of Limitless from the EU, UK, Brazil, Israel, South Korea, Turkey and China with a 19 December 2025 data-export deadline. Power: ESP32-S3 Wi-Fi average current ~120 mA per Espressif module measurements and community reports. Cognitive: irrelevant speech effect literature on verbal working memory.

## Revision history

**2026-08-08** — Second pass after an external review of the initial commit. Four corrections, all of them things the first version got wrong rather than merely omitted: the power budget conflated Wi-Fi and BLE regimes and overstated battery life by 3–4×; the README presented speaker diarization as if it solved addressee detection and claimed "no false positives"; pre-generation was described without acknowledging that it generates against incomplete context; and the legal section missed GDPR Chapter V transfers and the Meta/Limitless EU withdrawal. A new risk (cognitive interference) was added. Validation phase order was swapped.
