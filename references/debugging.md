# Audio debugging & measurement

You cannot hear your output — but you can measure it, which is often better. This file is the methodology that turned vague listener reports ("it crackles", "held notes are weird", "the strings feel quiet", "a weird pulsing drone at 1:02") into numbers, fixes, and verified improvements.

## The offline test harness

`node-web-audio-api` (npm, Rust-backed) runs the same Web Audio code headless with `OfflineAudioContext`. Port the engine once (parser + synth functions + graph; no scheduler needed offline — schedule everything, small slices only if the graph is heavy), render to WAV, analyze in Python/numpy.

**Render wall-clock vs realtime is the crackle predictor.** If a server core renders slower than realtime, a browser's shared audio thread has no chance. Worked numbers: dense MIDI at 32.4 s wall-clock for a 20 s slice (1.6×, crackled in browser) → after node diet → 13.1 s (0.65×, played cleanly). Re-measure after every optimization; this is the regression test.

Add env-var switches to the harness for **time window** (T0 + duration) and **channel filter** — that turns it into a stem renderer.

## Stem analysis (relative balance)

To answer "is instrument X too quiet": render each family solo through the *identical* chain, then compute per stem:
- **RMS → dBFS** (level)
- **Spectral centroid** (perceived brightness — a dull voice at equal RMS still sounds quieter; measure both)

Worked case: listener said the string ensemble felt under-emphasized. Stems showed ensemble −34.2 dBFS vs organ −19.1 (15 dB gap!) *and* the dullest centroid (1253 Hz vs 2066 Hz). Diagnosis: strings had inherited a pad gain (0.075) while playing a counter-melody role, plus a dark 1400 Hz lowpass.

**Balance targets that worked:** lead sits on top of its backing (0 to +1 dB over the accompaniment, brighter centroid does the rest); counter-melody sits 3–4 dB under the accompaniment; fix by adjusting the *loudest* offender down as well as the quiet one up.

**Compression is not a balance tool.** A compressor cannot know which instrument deserves prominence; it squashes whoever is loudest moment-to-moment, and on a master bus it inverts dynamics (see mixing.md failure mode 2). Level errors want level fixes — faders, informed by stem measurement.

## Amplitude-modulation measurement (pulsing/throbbing)

For "pulsing" complaints: extract the amplitude envelope of the suspect stem (|signal| smoothed ~50 ms), FFT the envelope, read the 0.2–2 Hz band relative to mean level = AM depth. Worked case: held distortion-guitar chord showed 6.3% AM depth + 18.6% sub-bass energy share; after the amp-chain fix (see midi-file-playback.md) 2.3% and 7.4%.

## Parser interrogation (localizing by symptom)

When a listener reports a problem at a timestamp, query the parsed note data before touching audio: filter notes by time window, duration threshold, channel. Worked case: "weird drone from ~1:02" → `notes with dur>3s` returned exactly three 5.5 s distortion-guitar power chords at 62.9 s / 171.9 s / 281.0 s — matching every reported occurrence and identifying the instrument the piano roll was too small to disambiguate. Also test pairing hypotheses offline (LIFO vs FIFO note matching, overlapping same-pitch notes) with a reference parser (e.g. python-mido) before blaming the file or the synth.

## The loop

symptom (listener's ear) → localize (parser query / stem isolation) → hypothesize (physics: beating detune, intermod difference tones, envelope shape, node count) → measure (harness) → fix → **re-measure the same metric** → ship. Never ship on "should be better now" when a number is available for a few seconds of rendering.

## Physics cheat-sheet for common symptoms

- Sub-1 Hz throb on held low notes → detuned unison beating (12 cents at 82 Hz ≈ 0.57 Hz). Fix: no detune on bass/distorted voices.
- Low "woofy" drone under distorted intervals → difference tones from clipping. Fix: highpass *before* the clipper (real amp topology).
- Held notes fading mid-note → percussive envelope on a sustained instrument (envelope taxonomy, midi-file-playback.md).
- Drums vanish only in dense sections + pumping → master compressor in constant gain reduction (mixing.md failure mode 2).
- Crackle then silence → too many live nodes (mixing.md failure mode 1) or render budget exceeded (measure wall-clock).
