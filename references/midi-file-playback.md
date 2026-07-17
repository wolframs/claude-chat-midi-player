# MIDI file playback

Turn the composition player into a general .mid player: hand-rolled SMF parser + synthesized GM instrument bank + transport with seek. All code battle-tested against a 22-track, 11k-note, 256 BPM FF8 boss MIDI.

Contents: parser · data extraction · GM voice mapping · GM drum map · seek architecture · UI notes

## SMF parser (complete, known-good)

Parse format 0/1: `MThd` (format, ntracks, ticksPerQuarter), then per `MTrk`: variable-length deltas, **running status** (status byte reused when high bit unset — forgetting this breaks most real files), meta events (only 0x51 tempo matters; skip the rest by length), sysex skipped by length, channel messages by high nibble: 0x9 note-on (velocity 0 = note-off!), 0x8 note-off, 0xB CC, 0xC program change, 0xE pitch bend (14-bit, center 8192).

Merge all tracks, stable-sort by tick, then build a tempo map: cumulative segments `[tick, secondsAtTick, usPerQuarter]`; `tsec(tick)` converts via the containing segment (`sec/tick = usPerQuarter / 1e6 / tpq`). Convert everything to seconds at parse time — the runtime engine then schedules in plain seconds.

The full parser is ~50 lines; reproduce it from the template in this skill's sibling implementation rather than reinventing: DataView + manual varint reads, no dependencies.

## Data extraction while walking events

- **Note pairing**: per (channel<<8|note) stack; note-on pushes [t, vel], note-off pops and emits [t, dur, note, ch, gain, program]. Flush unclosed notes at EOF with dur 0.3. **Channel 10 (index 9) drums emit on note-ON directly** — many files omit drum note-offs.
- **Velocity × CC7**: keep a per-channel CC7 timeline; final per-note gain = `(vel/127) × (cc7At(noteStart)/127)`, computed at parse time.
- **Pitch bends**: per-channel sorted list of [t, cents] (`value/8192 × 200` for the ±2-semitone default). At voice creation, binary-search the last bend ≤ noteStart, apply as initial detune offset, then schedule `detune.setValueAtTime` for every bend inside the note window. Store each oscillator with its base detune so bends add rather than replace.
- **Program at note-on** decides the voice family (program >> 3 = GM family 0–15).

## GM voice mapping (synthesized band)

**The envelope taxonomy — the single most important lesson:** instruments are either *percussive* (attack → exponential decay: keys, plucked guitar, bells, drums) or *sustained* (attack → hold at ~0.8 peak until note-off → short release: organ, strings, brass, reed, pipe, leads, bass, distortion guitar). Using a percussive envelope on a sustained instrument makes held notes audibly die halfway — this was heard by a listener before it was found in code. Write two envelope helpers and pick per family.

| Family (prog) | Recipe | Peak gain |
|---|---|---|
| bass 32–39, 87 | 2 saws ±6c → resonant LP env (200→900→250 Hz), sustained | 0.30 |
| organ 16–23 | sines at 1×, 2× (0.55) + saw 0.28 for rock organ 16–19, sustained | 0.09 |
| dist. guitar 29–31 | **1 saw, no detune** → per-note sustain env → shared amp bus | 0.40 into bus |
| clean guitar 24–28, pizz 45 | saw → LP sweeping 3000→500, percussive decay ≤1.5 s | 0.16 |
| strings/ens/pad 40–44, 48–55, 88–95 | 2 saws ±8c → LP 2000, attack min(0.14, 0.4·dur), sustained | 0.19 |
| brass 56–63 | 2 saws ±6c → LP env 500→2800→1200, sustained | 0.12 |
| reed 64–71 | square → LP 2500 + shared vibrato, sustained | 0.10 |
| pipe/flute 72–79 | sine + 0.3 triangle + shared vibrato, sustained | 0.15 |
| lead 80–86 | saw pair (square for 80) → LP 1200→4200 + vibrato, sustained | 0.19 |
| chromatic 8–15, fx 96+ | FM bell (mod ratio 2.76, decaying index) | 0.15 |
| keys 0–7 | FM EP (mod ratio 3.01), percussive | 0.16 |

**Shared per-channel infrastructure** (the node diet that took a dense file from 1.6× slower-than-realtime to 0.65×): one StereoPanner per channel (`pan = ((ch·7)%11−5)/14`), one vibrato LFO per channel (5.2 + ch·0.07 Hz, ±4.5 c), one guitar amp bus. Never create these per note.

**Guitar amp bus** (fixes the "pulsing drone" on sustained power chords): `notes → highpass 110 Hz → tanh(2.2) waveshaper → lowpass 2200 → gain 0.16 → master`. The pre-clip highpass is essential: clipping an interval generates sum/difference tones, and low difference tones are sub-bass mud; per-note detune adds a <1 Hz beat that hard clipping turns into a throb. Real amps roll off lows before distortion for exactly this reason. Measured effect of this chain: sub-bass share 18.6%→7.4%, amplitude-modulation depth 6.3%→2.3% on a held chord.

Skip notes with effective gain < 0.035; skip reverb sends for notes < 150 ms.

## GM drum map (channel 10, note → synth)

35/36 kick · 38/40/39 snare · 42/44 closed hat · 46 open hat · 49/52/55/57 crash · 51/53/59 ride (short 8 kHz noise) · 41–50 toms (pitch `90+(n−41)·14` Hz with 1.6× drop) · everything else: short band-passed tick. Use the drum synths from instruments.md, scaled by velocity gain.

## Seek architecture

On seek to time T: **tear down and rebuild the whole audio graph** (~50 ms, imperceptible, and far simpler than silencing in-flight nodes), then:
1. `startT = ctx.currentTime + 0.2 − T`
2. Binary-search the sorted event list for the first event ≥ T; set the scheduler pointer there.
3. **Backfill mid-flight notes**: scan for notes with `start < T && start+dur > T`; fire them immediately with their *remaining* duration at ~0.9 gain — seeking into a held chord lands inside the chord, not into silence.

Transport rules: pause = `ctx.suspend()` (free, clock freezes); plain resume = `ctx.resume()` (no rebuild); seek-while-paused just stores the position and draws the playhead — the rebuild happens on play. Rewind = seek(0).

## UI notes

File input: `<input type="file" accept=".mid,.midi">` → `arrayBuffer()` → parse locally; nothing leaves the machine (say so in the UI). Piano roll for 10k+ notes: **render once to an offscreen canvas**, per frame just `drawImage` + playhead line — redrawing thousands of rects at 60 fps is its own performance bug. Color per channel from a 12-color palette; auto-build the legend from channels present, labeled by GM family name. Show note/drum counts and duration after parse.

Known v0.2 limits (state them honestly when relevant): no CC64 sustain pedal, no CC91 per-channel reverb depth, synthesized timbres are impressionistic ("our band covers the file"), and extremely dense passages may still need voice-stealing. SoundFont mode (WebAudioFont / midi-js-soundfonts via jsdelivr/unpkg, or SpessaSynth) slots in as an alternative instrument layer behind the same parser + scheduler if fidelity ever beats charm.
