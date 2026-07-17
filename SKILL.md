---
name: webaudio-midi-composer
description: Compose original music and render it as an interactive in-chat player with real synthesized drums, piano-roll visualization, and a full Web Audio effects chain — and play back user-provided .mid files with a hand-rolled SMF parser plus a synthesized General MIDI instrument bank. Use this skill whenever the user asks Claude to compose music, write a MIDI track, make a song/theme/jingle, build a music or MIDI player in chat, play/open/render a .mid file, sonify data, or create any audible musical output — even if they don't say "MIDI" or "Web Audio". Also use it when iterating on a previously generated track or player (mix fixes, balance complaints, crackling/performance problems, weird-sounding instruments, new sections, playback controls).
---

# Web Audio MIDI composer (v0.2)

Compose music as note-event data and render it inside a chat widget (or HTML artifact) using the Web Audio API — no samples, no external files, no MIDI plumbing. Everything is synthesized: drums included.

## Why this skill exists

Naive attempts fail in two characteristic ways, both discovered the hard way:

1. **Scheduling everything upfront kills the audio thread.** A 100-second piece is ~1500+ events; instantiating every oscillator/filter/gain at t=0 creates thousands of live nodes → crackling, then total silence after a few seconds. **A lookahead scheduler is mandatory**, not an optimization.
2. **A master compressor with mixing-compressor settings inverts your dynamics.** Sustained layers (pads, drones, gallop bass) park it in constant gain reduction and duck the drum transients into inaudibility — precisely in the dense sections that should hit hardest, while sparse sections sound fine. Configure it as a safety limiter instead, and do balance with gain staging.

Both fixes are built into the template. Do not "simplify" them away.

## Workflow

1. **Read `references/player-template.md`** — the complete, working widget scaffold (UI, piano roll, lookahead scheduler, effects chain). Start from it; replace the composition data, keep the engine.
2. **Compose first, code second.** Read `references/composition.md` and plan on paper (in thinking): sections, tempo map, chord table, motif, where contrast lives. Only then encode it as event arrays.
3. **Pick instrument recipes** from `references/instruments.md` — synthesis for bass, lead, drums (kick/snare/hats/toms/timpani), FM bells, celesta, pads, stabs, risers, sub-drops.
4. **Mix by the rules** in `references/mixing.md` — gain staging table, stereo placement, effects sends, and the limiter settings. This file also documents the two failure modes above in detail.
5. **Render** as an inline widget (visualizer tool) or an HTML artifact. Same code works in both; the template notes the small differences.
6. **For .mid file playback**, read `references/midi-file-playback.md` — SMF parser, GM voice bank with the envelope taxonomy, seek architecture, drum map. Build a player the user loads their file into; never embed a copyrighted work's note data in the widget.
7. **When something sounds wrong** (crackles, imbalance, pulsing, dying notes), read `references/debugging.md` and measure before guessing — offline rendering with node-web-audio-api turns listener reports into numbers.

## Core data model

Music is arrays of events in **beats** (never seconds — the tempo map owns time):

```js
notes:  [beat, durationBeats, midiNote, ...flags]   // per melodic track
drums:  [beat, kind]                                 // 'k','s','hh','oh','t1','t2','tp','sr','cr'
chords: chordAt[bar] = chordId                       // lookup into a chord table
```

A tempo map `SEG = [[startBeat, bpm], ...]` plus `bt(beat)→seconds` / `tb(seconds)→beat` converts everything at schedule/draw time. This is what makes ritardandos, tempo jumps, and honest piano-roll x-positions cheap.

## Non-negotiables (v0.1)

- Lookahead scheduler (~0.6 s window, 120 ms pump interval), events sorted by beat, `try/catch` around each `fire()` so one bad node never stalls the pump.
- Master chain: `master gain → compressor(as limiter) → destination`; drums through a `tanh` waveshaper bus; one shared convolver reverb (generated impulse) and one tempo-synced feedback delay as sends.
- Pause = `ctx.suspend()` (freezes the clock, scheduler pauses for free); stop/end = `ctx.close()` and reset.
- Piano roll on `<canvas>`, colors must work in light and dark mode, playhead via `requestAnimationFrame`.
- No external assets, no `localStorage`, no libraries — pure Web Audio + canvas.

## Scope of v0.2

In: composition → synthesis → mix → inline player (play/pause, volume, click-to-seek, rewind, section labels); .mid file playback with GM band, pitch bends, CC7 volume, tempo maps; measurement-driven debugging methodology.
Not yet (planned): MIDI file *export* of composed pieces (event arrays are already MIDI-shaped — pure serialization), per-track mute/solo, CC64 sustain, loop mode, optional SoundFont instrument layer (WebAudioFont/SpessaSynth) behind the same parser+scheduler.
