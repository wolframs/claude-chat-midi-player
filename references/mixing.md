# Mixing, gain staging, and the two failure modes

Read this before touching any gain value or "simplifying" the engine.

## Failure mode 1: upfront scheduling (crackle → silence)

Symptom: audio crackles, then goes fully silent after a few seconds; UI keeps running.
Cause: instantiating every note's nodes at t=0. A ~100 s piece ≈ 1500 events ≈ 2500+ live AudioNodes; the audio render thread degrades (crackle) and then gives up (silence).
Fix (mandatory): lookahead scheduler — sorted event list, pointer, `setInterval(pump, 120)` instantiating only events within the next 0.6 s. Peak live nodes drop from thousands to ~40. Bonus: `ctx.suspend()` freezes `ctx.currentTime`, so pause needs zero extra logic.
Hardening: wrap each `fire()` in try/catch — an exception before `evPtr++` otherwise retries the same event forever and permanently stalls all playback.

## Failure mode 2: the compressor that inverts your dynamics

Symptom: drums audible in sparse sections, "filtered away to oblivion" in dense ones; overall pumping/choppy feel. The arrangement's density curve plays back inverted — big sections sound smallest.
Cause: a master DynamicsCompressor at mixing settings (e.g. threshold −16, ratio 5). Sustained layers (pads, drone, gallop bass, stabs) hold it in constant heavy gain reduction; sustained energy wins over transients, so kick/snare/hats get ducked exactly when the mix is fullest. Pumping = every kick triggering bus-wide gain reduction.
Fix: configure the compressor as a **safety limiter**, not glue — threshold −9, knee 4, ratio 5, attack 0.002, release 0.12 — and keep the summed level low enough that it rarely engages. Do all balance with gain staging, none with the compressor.

## Failure mode 3: percussive envelopes on sustained instruments

Symptom: held notes on some instruments audibly fade to silence mid-note; short notes sound fine.
Cause: one envelope helper (attack → exponential decay) used for everything. Sustained families (organ, strings, brass, reed, pipe, lead, bass, distorted guitar) need attack → hold (~0.8 peak) until note-off → short release. Keep the percussive envelope for keys, plucks, bells, drums. See the envelope taxonomy in midi-file-playback.md.

## Failure mode 4: detune + shared distortion = pulsing drone

Symptom: a "weird pulsing drone" under sustained low or distorted material.
Cause: detuned unison pairs beat (12 cents at 82 Hz ≈ 0.57 Hz — inaudible on short notes, a throb on held ones), and hard-clipping intervals generates sum/difference tones including sub-bass mud; together they produce a slow amplitude-modulated growl.
Fix: no detune on bass/distorted voices; highpass ~110 Hz *before* any waveshaper (real guitar-amp topology). Measured effect: sub-bass share 18.6%→7.4%, AM depth 6.3%→2.3%. Full analysis method in debugging.md.

## Gain staging table (reference mix, master 0.75 pre-limiter)

| Layer | Peak gain | Notes |
|---|---|---|
| Kick | 1.0 (+0.16 click) | via drumBus 1.15 → tanh shaper |
| Snare | 0.55 (+0.45 body) | big variant 0.7 |
| Hats | 0.17 closed / 0.19 open | panned +0.25 |
| Timpani | 0.8 | intro/transitions only |
| Bass saws | 0.30 accent / 0.21 | + sub sine 0.20 |
| Lead | 0.18 | quiet/call variant 0.08 |
| Harmony | 0.09 | panned −0.25 |
| Stabs | 0.11 | short — can afford presence |
| Arp | 0.075 accent / 0.048 | panned ±0.35 alternating |
| Pad (per tone) | 0.028 | felt, not heard |
| Drone (sum) | 0.11 | intro/calm only |
| Reverb return | 0.55 | one shared convolver |
| Delay return | 0.4, feedback 0.3 | tempo-synced dotted 8th |

Rule of thumb: transient sources get high peaks (they're gone in 200 ms); sustained sources get tiny gains (they're always there). If drums feel weak, lower the sustained layers before raising the drums.

## Stereo placement

Mono mixes sound flat regardless of composition quality. Reference placement: kick/snare/bass/lead dead center (the skeleton), hats +0.25, arp ping-pong ±0.35, pads tones alternating ±0.35, harmony −0.25, stabs +0.15, bells random wander, celesta alternating ±0.3. Guard `ctx.createStereoPanner` existence; fall back to a pass-through gain.

## Effects discipline

- One shared convolver (generated 1.3 s mono exponential-decay noise impulse) used via per-voice sends — never one reverb per note.
- One feedback delay, retimed at tempo boundaries with `setValueAtTime`.
- Drum bus → `tanh(x·1.8)` waveshaper: saturation glues and limits peaks. Melodic content stays clean (saws through a shaper turn to mush).
- Reverb sends: big on sparse/atmospheric (calls 0.8, celesta 0.75, timpani felt 0.6), small on dense rhythmic (snare 0.25, lead 0.2). Reverb on everything = distance on everything.

## Node hygiene

- Route linearly: source → filter → gain → (pan) → bus. Build the chain once per note, top down. Never connect-then-disconnect to rewire — a `disconnect(node)` on a connection that doesn't exist throws in spec-compliant browsers, and inside the pump that's failure mode 1's evil twin.
- Every source gets `start(t)` AND `stop(t+dur+tail)` — unstopped oscillators leak.
- All numbers displayed to the user go through rounding; all times through the tempo map.

## Measuring instead of guessing

Every balance decision above can and should be verified with stem rendering + RMS/centroid measurement — the full harness and worked case studies (including the balance targets: lead on top, counter-melody 3–4 dB under accompaniment) live in debugging.md. Compression is never the answer to a balance complaint.
