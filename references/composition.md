# Composition guide

How to compose so it lands for a listener, and how to encode it. Plan the whole piece before writing a single event array.

## 1. Contrast is the actual instrument

Loud only reads as loud next to quiet; fast next to slow. Design the *density curve* first: which sections are full, which are stripped, and where the drops and swells sit. A piece that is uniformly intense is uniformly boring. Concretely: if you want a huge phase-two, precede it with the sparsest section in the piece. Silence between unison hits is louder than the hits.

## 2. One motif, many costumes

Pick a short motif (3–5 notes) with one distinctive interval — e.g. a half-step sting for menace (E–F–E–B in Phrygian). Then re-state it in different emotional registers instead of inventing new material:
- whispered/foreshadowed (quiet lead, drowning reverb, before the piece proper starts)
- stated (full lead)
- lied about (same contour in the parallel major on celesta — "false calm"; the lie is one accidental wide)
- escalated (transposed up a half-step, octave-doubled, faster tempo)

Leitmotif economics: one idea recontextualized five times beats five ideas stated once, and it is exactly what makes a piece feel *composed* rather than generated.

## 3. Structure template (adapt, don't copy blindly)

Section (bars) — role:
- Intro (6–8) — slower BPM, drone + bells + timpani accelerando + riser + roll into the downbeat
- A: riff (8) — establish groove, no lead
- B: theme (8) — motif stated
- C: answer/ascension (8) — second phrase, higher register, harmonized second half
- Breakdown (4) — half-time, heavy
- False calm (4) — near-silence, major-mode quote, ritardando
- Phase two (8–12) — modulate up a semitone, tempo up ~10 BPM, theme restated octave-doubled
- Finale (4) — unison hits with silence, a scale run, one held high note, final chord
Ending on the parallel major (Picardy third) after sustained minor is a reliable emotional payoff.

## 4. Tempo map

`SEG=[[startBeat,bpm],...]`. Use it for: slow intro, ritardando into the calm (drop ~30 BPM for 2 bars), tempo jump at phase two. Retune the delay time at each boundary (`0.75*60/bpm` for dotted-eighth). Everything downstream (durations, piano-roll x, playhead) must go through `bt()/tb()` — never hardcode seconds.

## 5. Harmony quick kit

- Menace: Phrygian (♭2). The bass sliding root→♭2 at bar ends carries most of it.
- Drama dominant: major V borrowed from harmonic minor (B7 in E minor). Tritone bells (F against B in E) for intro dread.
- Chord table format: `[bassRoot, arpTones[4], stabVoicing[3]]`, progressions as `chordAt[bar]=id`.
- Modulating up a semitone for phase two: transpose the whole chord set +1 and reuse the theme arrays with `n+1`.

## 6. Harmonization pitfalls (learned the hard way)

Fixed-interval harmony (always −3 or −4 semitones) produces wrong notes against some chords (e.g. a minor 3rd below the root of a minor chord lands on its major 6th; a semitone against a major chord's root is mud). Options in order of safety:
1. Octave doubling (−12): always correct, sounds huge, the metal answer.
2. Hand-picked thirds per note, checked against the chord of that bar.
3. Fixed offset only after verifying every note it touches.
Never ship option 3 unverified.

## 7. Encoding patterns

- Gallop bass: per-bar pattern of `[offset, dur, transposeFlag, accent]` tuples; transposeFlag selects root / octave / ♭2. Loop it over the riff bars, swap in each bar's root.
- Arp: 16ths cycling a tone sequence like `[0,2,1,3,2,0,3,1]`; accent every 3rd sixteenth for a 3-against-4 cross-rhythm; alternate pan L/R per step; open the lowpass gradually with global progress.
- Themes: literal arrays `[offsetBeats, dur, midiNote]` relative to the section start, pushed with the section's beat offset. Verify each note against `chordAt` of its bar (chord tones on strong beats; passing tones on weak).
- Fills: every 4th bar, last beat — snare 16ths + hi/lo tom + snare. Big roll (2 beats of crescendoing 16ths) before section boundaries.
- Humanize: velocity ×(0.85+rand·0.3) everywhere; ±3.5 ms timing jitter on hats/arp only. Never jitter kick/snare/bass — the groove skeleton stays rigid.

## 8. Register discipline

Keep lanes clear: sub+bass below MIDI ~48, stabs/pads 48–77, lead 69–90, bells/celesta above 70. When everything fights for 60–80, nothing is audible (see mixing.md). The piano roll makes lane violations visible — look at it.
