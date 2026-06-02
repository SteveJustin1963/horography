
# issues that will stop it from working

- `analogRead` is too slow without DMA/timer; at default Teensy 4.1 settings you won't reliably catch the sharp transient peaks of a watch tick — you'll miss the peak 
amplitude entirely.
- Single-threshold peak detection in `loop()` is blocking but also non-deterministic: `analogRead` + `micros()` jitter plus no fixed sample rate means beat-to-beat 
interval measurements are noisy at the microsecond level you need.
- The `duration > 100000` (100 ms) refractory period is wrong for higher-frequency movements: a 28,800 BPH (8 bps) watch beats every ~125 ms, so 100 ms will swallow 
real beats on fast movements.
- `isTick = !isTick` assumes every detection alternates perfectly tick/tock, but double-pulses (drop + lock) inside one beat will toggle incorrectly and corrupt 
T1/T2.
- 12-bit ADC resolution on the Teensy 4.1 is fine, but the input pin is 3.3 V only; the piezo with a 1M bleeder can easily produce >3.3 V spikes (especially 
when bumped) and **destroy the ADC pin** — no clamping diode, no TVS, no series resistance to limit current into the pin.
- The 100nF coupling cap + 1MΩ input forms a high-pass filter with f_c ≈ 1.6 Hz, which is fine, but the 10k in series (R2) with the ADC sample-and-hold creates a 
significant source impedance — the Teensy 4.1 ADC wants <10k source impedance for accurate reads, otherwise readings droop during sampling.
- No op-amp shown in the main circuit (the "Pro-Tip" mentions TL072/LM358) — a raw piezo into a logic-level ADC will not exceed threshold 300 reliably. LM358 is also 
a bad choice here: it's slow (slew rate ~0.3 V/µs), noisy, and not rail-to-rail.
- No anti-aliasing filter before the ADC. If you later switch to DMA-based sampling (which you must), you'll get aliasing of mains hum and switching noise directly 
into your FFT/peak data.
- The 1MΩ bleeder is correct for piezo, but combined with the 100nF cap the time constant is 0.1s, so the DC bias settles slowly and the first second of readings will 
be garbage.
- TP4056 + MT3608 boost: the MT3608 has a ~250 kV/µs switching node and **no filtering shown** — that switching noise will inject directly into your analog audio 
line. No LC filter on the boost output, no ferrite, no analog supply rail. The OLED and MCU will also pollute the 5V rail.
- Grounds: the schematic ties everything to a single "GND" but doesn't show a star ground. With a boost converter and a µV-level audio amp in the same box, this 
guarantees 50/60 Hz hum coupling into the piezo line.
- The rotary encoder uses no pull-ups or debouncing in code/hardware; mechanical encoders will produce false triggers.
- No I2C pull-ups shown for the SSD1306 — many cheap OLED modules don't include them, and the Teensy 4.1's internal pull-ups are weak on I2C, causing intermittent 
display lockups.
- `analogReadResolution(12)` is Teensy-specific; on an Arduino Nano ESP32 it's a no-op or behaves differently — the code claims to be portable but isn't.
- Rate calculation divides by `targetInterval` then multiplies by 86400, but doesn't account for the BPH being configurable (18k/21.6k/28.8k) — hardcoded to 21,600 
only.
- Beat error formula `abs(t1 - t2) / 2000.0` is correct only when T1 and T2 are both full beat intervals; the comment "convert micros to ms" is wrong — dividing by 
1000 converts µs→ms, then `/2` averages it. The code has an extra `/2` baked into the 2000.
- No watchdog, no error handling — if the sensor disconnects, `loop()` will spin forever printing nonsense.
- Battery math: 1000 mAh at 3.7V boosted to 5V with an MT3608 (~85% efficient) running a Teensy 4.1 (often 100–300 mA) + OLED will **not** last 8–10 hours; real 
runtime is closer to 2–3 hours.
- Calibration warning is valid but unaddressed: no TCXO, no GPS reference built in, no way to actually calibrate the crystal.
- The "shielded cable" advice is correct, but the schematic shows unshielded wiring — the prose contradicts the diagram.
- 1.3" SH1106 mentioned in the text, but SSD1306 used in the BOM/schematic — they're not 100% drop-in compatible at the init level.
- Amplitude calculation is described in the text but **completely absent from the code** — the device will only ever show rate and beat error.
- No mention of needing an interrupt or timer-based capture; `micros()` overflows after ~70 minutes and the unsigned subtraction `currentTime - lastTickTime` will 
silently produce wrong values across the rollover unless guarded.

Most critical to actually getting a reading at all: **add a real preamp, protect the ADC input, and replace `loop()` polling with a timer-driven DMA sampler at a 
fixed rate.** Without those three changes, the device will either fry the input pin, read only noise, or give garbage microsecond timing.


