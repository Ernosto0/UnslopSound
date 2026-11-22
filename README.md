# UnslopSound

**Make AI-built apps sound intentionally designed.**

UnslopSound is a lightweight, non-destructive Claude Code skill (`/unslopsound`) that reviews and refines existing short sound effects in games and interactive applications. It acts like an audio refactoring assistant: preserve good work, identify fixable weaknesses, make restrained local edits, and let your ears compare the results before anything touches your project.

> **Core principle:** Don't regenerate what you can refine. Don't add sound where silence works better.

---

## What It Does

- **Discovers** existing short SFX (`.wav`, `.mp3`, `.ogg`), design docs, and code play-sites in your repository without modifying files.
- **Understands** your project's aesthetic, cue roles (UI feedback, pickups, jumps, hits, transitions, notifications), and trigger frequency.
- **Audits** files using local `ffprobe` / `ffmpeg` measurements (duration, peak, clipping, RMS/loudness, leading/trailing silence, DC offset, channel balance) and separates objective measurements from subjective design hypotheses.
- **Proposes** a short, ranked list of targeted improvements — including `Keep as-is` and `Remove` as valid recommendations.
- **Refines** approved targets non-destructively into `.unslopsound-review/<run-id>/candidates/` with conservative DSP (dead-air trimming, micro-fades, headroom fixes, gentle EQ, tail shaping, modest pitch nudges) and loudness-matched playback levels.
- **Compares & Applies** via a self-contained local HTML A/B listening page (`compare.html`, including an optional Blind Mode toggle) and only overwrites project assets after your explicit per-file approval.

## What It Does NOT Do (v0.1)

- Does **not** generate new sound effects or background music by default.
- Does **not** edit songs, compose soundtracks, build adaptive music systems, or clean up voice/dialogue.
- Does **not** bundle sound libraries, call paid APIs, upload your assets, or require a backend or MCP server.
- Does **not** claim an audio file was AI-generated.
- Does **not** listen to your audio with human ears — see [Honest Limitations](#honest-limitations).

---

## Installation (Claude Code)

### Option 1: Install for a single project
Copy `skills/unslopsound/` into your project's `.claude/skills/` directory:

```bash
mkdir -p .claude/skills
cp -r path/to/UnslopSound/skills/unslopsound .claude/skills/unslopsound
```

### Option 2: Install globally for all projects
Copy `skills/unslopsound/` into your user Claude skills directory (`~/.claude/skills/unslopsound`):

```bash
mkdir -p ~/.claude/skills
cp -r path/to/UnslopSound/skills/unslopsound ~/.claude/skills/unslopsound
```

---

## Usage

Open your game or app repository in Claude Code and run:

```text
/unslopsound
The effects in this game feel generic and cheap. Review the existing audio,
suggest improvements, and refine a few of the worst examples.
```

You can also run an audit-only pass first:

```text
/unslopsound
Audit the existing UI and gameplay sound effects without editing anything yet.
```

---

## Prerequisites & Supported Inputs

- **Supported inputs:** Local `.wav`, `.mp3`, and `.ogg` short sound effects.
- **Optional (recommended) local tools:** `ffmpeg` and `ffprobe` on `PATH` for full technical measurement and non-destructive audio refinement.
  - **Windows:** `winget install Gyan.FFmpeg`
  - **macOS:** `brew install ffmpeg`
  - **Debian / Ubuntu:** `sudo apt install ffmpeg`
- **Without FFmpeg:** `/unslopsound` still runs read-only discovery, code-usage mapping, and basic audit reporting, then tells you how to install FFmpeg without modifying your machine or assets.

---

## Honest Limitations

- **No native ear:** Unless your environment provides audio-understanding capability, Claude does **not** hear timbre or aesthetics directly. UnslopSound relies on deterministic file measurements (`ffprobe`/`ffmpeg`) plus your codebase context, and clearly separates measured facts from design hypotheses. **Your ears are the final quality gate.**
- **Local DSP boundaries:** Conservative FFmpeg filters can fix laggy leading silence, start/end clicks, excess tails, loudness outliers, and mild harshness, but cannot undo severe baked-in hard clipping, heavy low-bitrate MP3 artifacts, or a fundamentally wrong sound metaphor.
- **Lossy formats:** Re-encoding already low-bitrate `.mp3` or `.ogg` files can introduce minor generation loss; UnslopSound warns you when source bitrates are low.
- **Review artifacts:** Refinement creates `.unslopsound-review/` in your project root. Keep it out of git (or delete it after you finish reviewing and applying changes).

---

## Reporting Poor Results

If `/unslopsound` gives unhelpful recommendations, breaks a file format, or produces worse-sounding candidates, please open an issue with:
1. Your OS, shell, and `ffmpeg -version` output.
2. The cue role (e.g., UI click, pickup, hit) and source format/duration.
3. The relevant entry from `.unslopsound-review/<run-id>/changelog.json` (no proprietary audio needed).
4. What sounded wrong compared to the original.

---

## License

[MIT](LICENSE)
