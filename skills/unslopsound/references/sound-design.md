# UnslopSound Reference: Sound Design, Measurement & Conservative Repair

Use this reference during **Step 3 (Audit)**, **Step 5 (Refine)**, and **Step 6 (Compare & Apply)**. Keep edits minimal, reversible, and tailored to the game's existing sonic identity.

---

## 1. Cue-Type Design Matrix

| Cue Type | Typical Duration | Trigger Frequency | Spectral & Dynamic Guidance | Common "Generic / Slop" Traps |
|---|---|---|---|---|
| **UI Click / Toggle / Back** | 20–150 ms | Very high | Crisp onset (<10 ms leading silence), restrained level (-18 to -12 dB below primary action cues), smooth top end | Laggy dead air before attack, harsh 5–8 kHz snap, loud click fatigue |
| **UI Hover / Focus** | 15–60 ms | Extreme | Ultra-subtle or **silent**; strongest candidate for `Remove` | Repetitive ticking on every mouseover or list scroll |
| **Pickup / Coin / Collect** | 120–600 ms | Medium–High | Clear attack, controlled tail; avoid piling up long ringing tails | >800 ms resonant tail that muddies rapid pickups; piercing 6–9 kHz harmonics |
| **Jump / Step / Movement** | 80–350 ms | High | Warm/balanced body; avoid fatiguing 2.5–5 kHz glare | Loud, sharp transient on every step or jump; repetitive harshness |
| **Hit / Impact / Damage** | 100–500 ms | Medium | Clean transient punch with headroom (peak <= -1.0 dBFS); controlled sub-60 Hz rumble | Brickwalled clipping, +8 dB louder than sibling cues, abrupt tail cut |
| **Transition / Whoosh** | 200–900 ms | Low–Medium | Smooth fade-in and fade-out; no DC pop at boundaries | Abrupt cut-in/cut-out clicks, excessive low-end rumble |
| **Notification / Unlock** | 300–1500 ms | Low | More tonal room allowed, but still matched to the game's material palette | Jarring loudness jump compared to normal gameplay cues |

---

## 2. Measurable Defect Taxonomy & Thresholds

Always report these under **Measured Technical Findings** with exact values from `ffprobe`/`ffmpeg`.

| Defect | Inspection Signal | Flag Threshold | Conservative Repair |
|---|---|---|---|
| **Leading silence (laggy onset)** | `silencedetect` at `-45 dB` | >25 ms (UI, hit, jump) or >50 ms (other) | `silenceremove=start_periods=1:start_duration=0:start_threshold=-45dB` + 2–5 ms fade-in |
| **Excessive trailing silence** | `silencedetect` at `-50 dB` | >250 ms dead air at end | Trim tail (`atrim` or reversed `silenceremove`) + 10–25 ms fade-out |
| **Start/end click or DC offset** | `astats` (`DC offset`) or abrupt cut | `|DC offset| > 0.01` or audible/sharp boundary | `highpass=f=25` + micro-fades (`afade` 3–10 ms in, 8–20 ms out) |
| **Clipping / Zero headroom** | `volumedetect` (`max_volume`, `histogram_0db`), `astats` (`Peak level dB`, `Flat factor`) | `max_volume >= -0.2 dB` or `Flat factor > 0` | Reduce gain so peak <= `-1.0 dBFS`. **If severe flat-top clipping is baked in (`Flat factor` high), mark as partially/fully unrepairable.** |
| **Loudness outlier** | Active-region RMS (`astats` / `volumedetect` `mean_volume`) | >6 dB hotter or quieter than sibling cues in same tier | Gain trim toward project norm |
| **Channel imbalance** | `astats` per-channel `RMS level dB` | L/R RMS diff >3 dB on non-directional cue | Center balance or fold to mono (`pan=mono\|c0=0.5*c0+0.5*c1`) |
| **Excessive tail length** | `ffprobe` duration + decay profile | >700 ms on high-frequency pickup/UI cue | `atrim=0:<target_s>` + exponential/logarithmic `afade=t=out` |
| **Low-end mud / rumble** | Context + sub-bass energy on non-impact cue | Boomy UI/pickup cue | Gentle `highpass=f=60` to `f=100` (1st/2nd order) |
| **High-end harshness (hypothesis)** | Context + high-frequency cue role | Piercing chimes/clicks | Gentle `highshelf=f=6000:g=-3` or wide `equalizer=f=3500:t=q:w=1.2:g=-2.5` (cap at ±4 dB) |
| **Loop seam click** | Only for cues marked/used as looping | Amplitude/DC discontinuity at start vs. end | Short crossfade loop or report if asset is unsuitable for looping |

---

## 3. Inspection Commands (FFprobe & FFmpeg)

Always quote file paths and filtergraph strings properly for the active shell (double quotes work across both bash and PowerShell for standard `-af "..."` strings; avoid unquoted semicolons or pipes).

### 3.1 Stream & Format Metadata
```bash
ffprobe -v error -show_format -show_streams -of json "<input_file>"
```
Extract: `codec_name`, `sample_rate`, `channels`, `duration`, `bit_rate`, `bits_per_raw_sample`.

### 3.2 Single-Pass Technical Stats
```bash
ffmpeg -hide_banner -nostats -i "<input_file>" -af "volumedetect,astats=metadata=0:reset=0,silencedetect=n=-45dB:d=0.02" -f null -
```
Extract from stderr:
- `volumedetect`: `mean_volume`, `max_volume`, `histogram_0db`
- `astats`: `DC offset`, `Peak level dB`, `RMS level dB`, `Flat factor`, `Peak count`, per-channel `RMS level dB`
- `silencedetect`: `silence_start`, `silence_end`, `silence_duration`

### 3.3 Integrated Loudness (Only for Cues >= 1.0 s)
```bash
ffmpeg -hide_banner -nostats -i "<input_file>" -af "ebur128=peak=true" -f null -
```
Extract `I:` (Integrated LUFS) and `Peak:` (True Peak dBFS).

---

## 4. Short-SFX Loudness Matching Protocol

Fair A/B listening requires comparable playback loudness so louder is never mistaken for better.

1. **Why `ebur128` fails on short SFX**: ITU-R BS.1770 gating needs >=400 ms momentary / ~1 s integrated windows. On 40–300 ms clicks, jumps, and hits, `ebur128` often reports `-70.0 LUFS` or invalid values.
2. **Two-tier matching rule**:
   - **Duration >= 1.0 s** *and* valid `ebur128` (`I > -60 LUFS`): match **Integrated LUFS**.
   - **Duration < 1.0 s** (most SFX) *or* invalid `ebur128`: match **active-region RMS** (`astats` `RMS level dB` or `volumedetect` `mean_volume`).
     - *Important*: If you trimmed significant dead air from the original, measure the original's RMS on its **trimmed active region** (e.g., via a quick `-f null -` pass with the same `silenceremove`) so dead air does not artificially lower the original's reference RMS.
3. **Compute & Clamp Gain**:
   - Measure the intermediate candidate's `RMS level dB` and `Peak level dB` in 32-bit float via `-af "<base_filters>,astats=metadata=0:reset=0" -f null -` (never through a 16-bit WAV, which clips filter overshoots and hides true peak).
   - `gain_db = reference_rms_db - candidate_rms_db`
   - Check peak headroom: `projected_peak = candidate_peak_db + gain_db`
   - **Peak Safety Clamp**: If `projected_peak > -1.0 dBFS` (use `-2.0 dBFS` for lossy MP3/OGG encoders to absorb MDCT/codec peak overshoot), clamp `gain_db = target_ceiling - candidate_peak_db`.
   - Record the final `loudness_match_delta_db` (`candidate_final_rms - reference_rms`) in `changelog.json`. If clamping prevented an exact match (common when repairing a clipped original), state that clearly in the changelog and `compare.html`.

---

## 5. Ordered Conservative DSP Filter Chain

When combining edits in a single FFmpeg pass, keep filters in this order to avoid compounding artifacts:

1. **DC / sub-rumble cleanup**: `highpass=f=25` (or `f=80` for non-bass cues)
2. **Dead-air trim**: `silenceremove=start_periods=1:start_duration=0:start_threshold=-45dB` (and optional `atrim=0:<end_s>`)
3. **Protective micro-fades / tail shaping**: `afade=t=in:st=0:d=0.004,afade=t=out:st=<fade_start>:d=<fade_dur>:curve=exp`
4. **Restrained EQ** (keep boosts/cuts within ±4 dB): `highshelf=f=6000:g=-3`, `equalizer=f=3200:t=q:w=1.2:g=-2.5`, or `lowpass=f=12000`
5. **Modest pitch nudge** (only when justified, ±1–3 semitones): `asetrate=<sr>*<ratio>,aresample=<sr>` (note: slightly changes duration; keep ratio within `0.85–1.18`)
6. **Loudness match & peak safety**: `volume=<gain_db>dB`

### Format-Preserving Encoding Settings
Match the original file's container, sample rate (`-ar`), and channel count (`-ac`) unless explicitly changing channels:
- **WAV**: `-c:a pcm_s16le` (or match source `pcm_s24le` / `pcm_f32le` if detected)
- **MP3**: `-c:a libmp3lame -q:a 2` *(warn if source bitrate < 128 kbps due to lossy re-encode generation loss)*
- **OGG**: `-c:a libvorbis -q:a 5` *(warn if source bitrate < 96 kbps)*

---

## 6. When NOT to Edit

Always consider these three outcomes before running FFmpeg:

- **Keep as-is**: Clean onset (<25 ms), headroom <= -1.0 dBFS, zero clipping, reasonable duration and level for its cue role, no obvious technical faults.
- **Remove / silence**: High-frequency hover/scroll/tick cues that add clutter rather than feedback, duplicate cues, or unreferenced orphan assets.
- **Unrepairable with local DSP**: Severe baked-in hard clipping (flat-topped waveform), heavy low-bitrate metallic MP3 artifacts, excessive baked-in reverb you need to remove, or a fundamentally mismatched sound metaphor. State this plainly instead of over-processing.

---

## 7. Pre-Present Technical QA Checklist

Run `ffprobe` + the single-pass `ffmpeg` stats command on **every** generated candidate before showing `compare.html` to the user:

1. File exists, is non-empty, and decodes cleanly with `ffprobe`.
2. Container/extension, codec, and sample rate are compatible with the project's original asset.
3. Duration is > 0.01 s and matches the intended edit (no accidental over-trimming to silence).
4. `max_volume <= -1.0 dBFS` (or strictly lower than a clipped original).
5. Zero new clipping (`max_volume <= -1.0 dBFS`, `histogram_0db` absent, `Flat factor` not increased).
6. No excessive leading silence reintroduced.
7. Loudness delta vs. active original is within `±1.5 dB` (or peak-clamping reason is explicitly logged).
8. Original asset path is 100% untouched.
