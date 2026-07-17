# Instrument recipes

Copy-paste-complete synthesis functions. All assume the engine globals from player-template.md (`ctx, master, rev, dly, drumBus, noiseBuf, f, env, send, mkPan, hum`). Gains shown are the balanced values from the reference mix — change them only per mixing.md.

Contents: drums · bass · lead+harmony · bells/celesta · pads/stabs/drone · transitions

Envelope rule: every voice is either percussive (attack→decay) or sustained (attack→hold→release) — see the taxonomy in midi-file-playback.md and failure mode 3 in mixing.md before writing any new voice. The GM instrument bank (organ, guitar, strings-as-melody, brass, reed, pipe, GM drum map) also lives in midi-file-playback.md.

## Drums (the reason this skill exists)

Pure oscillators make sad clicks. Real percussion = pitch drops + filtered noise. All drums route to `drumBus` (→ tanh waveshaper → master) for glue and grit.

**Kick** — sine with fast pitch drop 160→42 Hz, plus a 1.1 kHz square click for beater attack:
```js
function kick(t){const o=ctx.createOscillator(),g=ctx.createGain();o.type='sine';
  o.frequency.setValueAtTime(160,t);o.frequency.exponentialRampToValueAtTime(42,t+0.11);
  env(g,t,0.002,1.0,0.24);o.connect(g).connect(drumBus);o.start(t);o.stop(t+0.32);
  const c=ctx.createOscillator(),cg=ctx.createGain();c.type='square';c.frequency.value=1100;
  env(cg,t,0.001,0.16,0.018);c.connect(cg).connect(drumBus);c.start(t);c.stop(t+0.05);}
```

**Noise hit** — the shared building block (snare body, hats, crash, rolls). Note the clean pan routing; do NOT "optimize" with disconnect tricks:
```js
function noiseHit(t,peak,dec,ftype,fq,q,dest,pan){const s=ctx.createBufferSource();s.buffer=noiseBuf;
  const fl=ctx.createBiquadFilter();fl.type=ftype;fl.frequency.value=fq;fl.Q.value=q||0.7;
  const g=ctx.createGain();env(g,t,0.001,peak,dec);
  s.connect(fl);fl.connect(g);
  let tail=g;
  if(pan&&ctx.createStereoPanner){const p=ctx.createStereoPanner();p.pan.value=pan;g.connect(p);tail=p;}
  tail.connect(dest||drumBus);
  s.start(t);s.stop(t+dec+0.1);return g;}
```

**Snare** = bandpass noise 1700 Hz + pitch-dropping triangle body 210→160 Hz; reverb send makes it big:
```js
function snare(t,big){const g=noiseHit(t,big?0.7:0.55*hum(),big?0.22:0.15,'bandpass',1700,0.8);
  send(g,rev,big?0.45:0.25);
  const o=ctx.createOscillator(),og=ctx.createGain();o.type='triangle';
  o.frequency.setValueAtTime(210,t);o.frequency.exponentialRampToValueAtTime(160,t+0.08);
  env(og,t,0.001,0.45,0.09);o.connect(og).connect(drumBus);o.start(t);o.stop(t+0.15);}
```

**Hats** = highpass noise. Closed: 7.5 kHz, 35 ms. Open: 6.8 kHz, 170 ms. Pan ~+0.25, jitter timing ±3.5 ms:
```js
noiseHit(t+(Math.random()-0.5)*0.007, 0.17*hum(), 0.035, 'highpass', 7500, 0.7, drumBus, 0.25) // closed
noiseHit(jt, 0.19, 0.17, 'highpass', 6800, 0.7, drumBus, 0.25)                                  // open
```

**Toms** — sine drops: hi 220→120, lo 150→80, 180 ms. **Timpani** — sine 80→41 Hz over 0.4 s, 0.9 s decay, plus a lowpassed 300 Hz noise "felt" layer with heavy reverb send. **Crash** — highpass noise 3.8 kHz, decay 1.4 s, reverb send 0.5. **Snare roll** — 16th-note noiseHits with peak scaled by roll progress (`0.06+0.42*i/N`).

## Bass

Two detuned saws (±7 cents) through a per-note resonant lowpass with an envelope (this filter motion IS the "bite"), plus a sine sub one octave down when the note is low enough:
```js
function bassNote(t,dur,n,ac){const lp=ctx.createBiquadFilter();lp.type='lowpass';lp.Q.value=4;
  lp.frequency.setValueAtTime(180,t);
  lp.frequency.linearRampToValueAtTime(ac?1500:750,t+0.02);
  lp.frequency.exponentialRampToValueAtTime(200,t+Math.max(0.1,dur*0.8));
  const g=ctx.createGain();env(g,t,0.005,(ac?0.3:0.21)*hum(),dur);lp.connect(g).connect(master);
  for(const det of[-7,7]){const o=ctx.createOscillator();o.type='sawtooth';
    o.frequency.value=f(n);o.detune.value=det;o.connect(lp);o.start(t);o.stop(t+dur+0.1);}
  if(n<48){const sub=ctx.createOscillator(),sg=ctx.createGain();sub.type='sine';
    sub.frequency.value=f(n-12);env(sg,t,0.005,0.2,dur);
    sub.connect(sg).connect(master);sub.start(t);sub.stop(t+dur+0.1);}}
```
Accent flag (`ac`) opens the filter twice as far — program accents on downbeats/pattern anchors.

## Lead + harmony

Lead: two saws ±6 cents, shared vibrato LFO whose depth blooms on held notes (2→9 cents over 0.8 s), lowpass opening 1.2→4.5 kHz at note start, sends to delay (0.28) and reverb (0.2). A `quiet` flag (gain 0.08, reverb 0.8) turns it into a "distant call" for foreshadowing.
```js
function leadNote(t,dur,n,quiet){const g=ctx.createGain();env(g,t,0.015,(quiet?0.08:0.18)*hum(),dur);
  const lp=ctx.createBiquadFilter();lp.type='lowpass';
  lp.frequency.setValueAtTime(1200,t);lp.frequency.linearRampToValueAtTime(4500,t+0.06);
  lp.connect(g).connect(master);send(g,dly,quiet?0.4:0.28);send(g,rev,quiet?0.8:0.2);
  const v=ctx.createOscillator(),vg=ctx.createGain();v.frequency.value=5.5;
  vg.gain.setValueAtTime(dur>1.2?2:1,t);vg.gain.linearRampToValueAtTime(dur>1.2?9:3,t+Math.min(dur,0.8));
  for(const det of[-6,6]){const o=ctx.createOscillator();o.type='sawtooth';o.frequency.value=f(n);
    o.detune.value=det;v.connect(vg).connect(o.frequency);o.connect(lp);o.start(t);o.stop(t+dur+0.15);}
  v.start(t);v.stop(t+dur+0.15);}
```
Harmony voice: single saw, gain 0.09, lowpass 2.8 kHz, panned −0.25. Safest intervals: octave below (always right, sounds huge) or hand-checked 3rds/4ths — see composition.md on harmonization pitfalls.

## Bells and celesta

**FM bell** ("cursed cathedral"): carrier at f(n), modulator at f(n)×2.76 (inharmonic ratio = bell), modulation index starting at f(n)×2.4 decaying to 1 over the duration. Reverb send 0.5, random pan per strike.

**Celesta** (the "false calm" instrument): plain sine + a quiet 3× partial that decays in 0.3 s, sharp attack, long release, reverb send 0.75, alternating pan. Innocence is mostly the absence of harmonics.

## Pads, stabs, drone

**Pad**: one saw per chord tone, detuned ±6 alternating, each through a 750 Hz lowpass, 0.35 s attack/release, tones panned alternately ±0.35. Gain per tone 0.028 — pads are felt, not heard; see mixing.md before raising this.

**Stab** ("brass"): 3-note low voicing, saws, lowpass sweeping 2600→400 Hz over the (short) duration, gain 0.11. Place on beat 1 and the syncopation points; stabs are punctuation.

**Drone**: 3 saws detuned ±11 at a low root (and optionally the fifth), lowpass slowly opening 120→900 Hz across the whole duration, 2 s fade-in. Intro/atmosphere only.

## Transitions

**Riser**: looped noise through a bandpass sweeping 300 Hz→9 kHz + a saw pitch-rising an octave, both crescendoing from silence over 4–8 beats into a downbeat. Duration must be computed through the tempo map (`bt(b+durBeats)-bt(b)`).

**Sub-drop**: sine 65→28 Hz over 0.6 s, gain 0.65 — place ~0.3 beats before a section slam. The cheap "oh no" signifier.

**Unison hit** (finale slams): stab + accented bass root + kick + short highpass crash, all at the exact same timestamp, with real silence between hits.
