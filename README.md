# webaudio-midi-composer

```
lead  |                        ██ █ █ ██ ██████
arp   |        █ █ ██ █ █ ██ █ █ ██ █ █ █
bass  |    ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ █████
drums |    █  █ █ ██ █  █ █ ██ █ ██ █ ██ █   █
      +--------------------------------------->
           the gallop        the climb    dawn
```

A [Claude Skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills) that teaches Claude to **compose original music** and **play MIDI files** directly inside a chat conversation — with real synthesized drums, a full Web Audio effects chain, a piano-roll visualization, and playback controls. No samples. No libraries. No audio files. Every sound is oscillators and filtered noise, born at play-time in your browser.

## What it does

**Compose.** Ask Claude for a boss battle theme, a lullaby, a jingle, a leitmotif-driven five-phase epic with a tempo map and a Picardy third — it plans the composition (structure, motif, harmony, density curve), synthesizes every instrument from scratch, mixes it by documented gain-staging rules, and renders an interactive player right into the chat.

**Play.** Hand it a `.mid` file and it builds a player with a hand-rolled Standard MIDI File parser and a synthesized General MIDI band: tempo maps, running status, per-channel volume (CC7), pitch bends, the GM drum map, click-to-seek on the piano roll. Files are parsed and synthesized entirely locally — nothing leaves the machine, and no song data is ever embedded in the widget.

**Debug.** The skill's crown jewel: a measurement methodology that gives Claude ears it doesn't have. Offline rendering as a test harness, per-instrument stem analysis, amplitude-modulation forensics — so "it crackles" and "the strings feel quiet" become numbers, fixes, and verified improvements.

## Why the scars are the point

This skill was vibed into existence in a single conversation and then debugged against an 11,000-note, 256 BPM Final Fantasy VIII boss MIDI. Every hard-won lesson is documented as a **failure mode** with symptom → cause → fix → measured result:

| # | Failure mode | The measurement that caught it |
|---|---|---|
| 1 | Scheduling all notes upfront → crackle, then silence | ~2,500 live audio nodes; fixed with a lookahead scheduler (~40 nodes) |
| 2 | Master compressor inverting the dynamics — drums vanish exactly in the *dense* sections | Reconfigured as a limiter; balance moved to gain staging |
| 3 | Percussive envelopes on sustained instruments — held notes fading mid-note | RMS traces showing exponential decay on held chords |
| 4 | Detuned unison + shared distortion = sub-bass pulsing drone | Sub-bass share 18.6%→7.4%, AM depth 6.3%→2.3% after adding a pre-clip 110 Hz highpass (real amp topology) |

Plus: a render-speed benchmark (offline wall-clock vs. realtime as a crackle predictor: 1.6× → 0.65× after the node diet), and stem-measured balance targets (lead on top; counter-melody 3–4 dB under accompaniment; **compression is never the answer to a balance complaint**).

A model following this skill inherits the scar tissue without the wounds.

## Structure

```
webaudio-midi-composer/
├── SKILL.md                          # triggering, workflow, non-negotiables
└── references/
    ├── player-template.md            # engine scaffold: scheduler, tempo map, graph, transport, roll
    ├── instruments.md                # synthesis recipes: drums, bass, leads, FM bells, pads, risers
    ├── composition.md                # the craft: contrast, motif economics, structure, harmony
    ├── mixing.md                     # gain staging, stereo, effects discipline, failure modes 1–4
    ├── midi-file-playback.md         # SMF parser, GM voice bank, envelope taxonomy, seek architecture
    └── debugging.md                  # the measurement methodology, with worked case studies
```

## Install

**Claude.ai** — Settings → Capabilities → Skills → upload the packaged `.skill` file (or open it in a chat and hit *Save skill*). Requires the code-execution/analysis capability for the composition workflow; the rendered players run in any chat.

**Claude Code** — drop the folder into your skills directory (e.g. `~/.claude/skills/webaudio-midi-composer/`).

## Use

> "Compose a boss battle theme with real drums"
>
> "Make me a 90-second synthwave track, piano roll and all"
>
> "Here's a .mid file — build a player and make it sound good"
>
> "The strings feel buried — measure it and fix the mix"

## Honest limitations (v0.2)

Synthesized GM is impressionistic — a rock organ built from four sines and a dirty saw is a *gesture* at a Hammond. No CC64 sustain pedal, no per-channel reverb depth, and extremely dense files may still want voice-stealing. Roadmap: MIDI *export* of composed pieces (the event arrays are already MIDI-shaped), per-track mute/solo, loop mode, and an optional SoundFont layer (WebAudioFont / SpessaSynth) behind the same parser + scheduler for when fidelity beats charm.

## Provenance

Built in one sitting by a human ear and a Claude with an FFT, iterating from "the drums are sad clicks" to a seekable GM synthesizer with a forensics manual. The first piece composed *for* this band — knowing exactly what every voice sounds like, because we tuned them together — was a credits theme called *Dawn Is Mandatory*.

The door is the scary part. The dawn is mandatory.