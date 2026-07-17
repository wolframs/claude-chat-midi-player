# Player template

The complete working scaffold. Replace the COMPOSITION DATA block; keep the ENGINE unless you know why you're changing it.

Contents: 1. Structure · 2. Full template code · 3. Widget vs artifact differences · 4. Extension points

## 1. Structure

```
UI (canvas piano roll + play button + time + legend + volume + section label)
COMPOSITION DATA (tempo map, chord table, event arrays)      ← per-piece
ENGINE (bt/tb time math, event merge+sort, synth functions,
        lookahead scheduler, audio graph setup, draw loop)   ← reusable
```

## 2. Full template code

The HTML shell (inline styles use host CSS variables so light/dark both work):

```html
<div style="border: 0.5px solid var(--border); border-radius: 12px; padding: 12px; background: var(--surface-1);">
  <canvas id="roll" style="width:100%; height:240px; display:block;"></canvas>
  <div style="display:flex; align-items:center; gap:12px; margin-top:10px; flex-wrap:wrap;">
    <button id="play" aria-label="Play" style="width:44px;height:44px;border-radius:50%;display:flex;align-items:center;justify-content:center;font-size:18px;"><i class="ti ti-player-play" aria-hidden="true"></i></button>
    <span id="time" style="font-size:13px;color:var(--text-secondary);font-family:var(--font-mono);min-width:92px;">0:00 / 0:00</span>
    <!-- legend spans per track, colored 9px squares -->
    <label style="font-size:12px;color:var(--text-secondary);display:flex;align-items:center;gap:6px;">vol <input type="range" id="vol" min="0" max="100" value="70" step="1" style="width:90px;"></label>
  </div>
  <div id="sect" style="font-size:12px;color:var(--text-muted);margin-top:8px;font-family:var(--font-mono);">&nbsp;</div>
</div>
```

The script (this is the engine from the reference implementation; composition data shown minimal):

```js
// ---- TEMPO MAP: [startBeat, bpm]. All musical time is in beats.
const TOTBEATS=224;
const SEG=[[0,96],[32,142],[152,108],[160,152]];
let segT=[];{let t=0;for(let i=0;i<SEG.length;i++){segT.push(t);
  const end=(i+1<SEG.length?SEG[i+1][0]:TOTBEATS);t+=(end-SEG[i][0])*60/SEG[i][1];}}
function bt(b){let i=SEG.length-1;while(i>0&&b<SEG[i][0])i--;return segT[i]+(b-SEG[i][0])*60/SEG[i][1];}
function tb(t){let i=SEG.length-1;while(i>0&&t<segT[i])i--;return SEG[i][0]+(t-segT[i])*SEG[i][1]/60;}
const TOTAL=bt(TOTBEATS)+1.6;   // tail for reverb/crash decay

// ---- CHORD TABLE: [bassRootMidi, arpTones[4], stabVoicing[3]]
const CH=[[40,[64,67,71,76],[52,59,64]] /* , ... */];
const chordAt=[]; // chordAt[bar] = index into CH

// ---- EVENT ARRAYS (built per composition; see composition.md)
let bass=[],arp=[],lead=[],harm=[],bells=[],cel=[],drums=[],drone=[],
    risers=[],pads=[],stabs=[],hits=[],calls=[],drops=[];

// ---- MERGE into one sorted list: [beat, type, ...payload]
let events=[];
for(const e of bass)events.push([e[0],'bass',e[1],e[2],e[3]]);
for(const e of lead)events.push([e[0],'lead',e[1],e[2]]);
for(const e of drums)events.push([e[0],'dr',e[1],e[2]]);
// ...one loop per array, then:
events.sort((a,b)=>a[0]-b[0]);

// ---- ENGINE
const f=m=>440*Math.pow(2,(m-69)/12);          // MIDI → Hz
const hum=()=>0.85+Math.random()*0.3;           // velocity humanization
let ctx=null,master=null,rev=null,dly=null,drumBus=null,
    startT=0,playing=false,noiseBuf=null,evPtr=0,schedTimer=null;

function mkPan(p){if(ctx.createStereoPanner){const x=ctx.createStereoPanner();x.pan.value=p;return x;}return ctx.createGain();}
function mkBufs(){ // shared noise buffer + generated reverb impulse
  const len=ctx.sampleRate*1.5;noiseBuf=ctx.createBuffer(1,len,ctx.sampleRate);
  const d=noiseBuf.getChannelData(0);for(let i=0;i<len;i++)d[i]=Math.random()*2-1;
  const rl=ctx.sampleRate*1.3,ir=ctx.createBuffer(1,rl,ctx.sampleRate);
  const dd=ir.getChannelData(0);for(let i=0;i<rl;i++)dd[i]=(Math.random()*2-1)*Math.pow(1-i/rl,2.5);
  rev.buffer=ir;}
function env(g,t,a,peak,dec){g.gain.setValueAtTime(0.0001,t);
  g.gain.linearRampToValueAtTime(peak,t+a);
  g.gain.exponentialRampToValueAtTime(0.0001,t+a+dec);}
function send(g,dest,amt){const s=ctx.createGain();s.gain.value=amt;g.connect(s).connect(dest);}

// Synth voices: see instruments.md for the full set
// (bassNote, leadNote, harmNote, bell, celesta, droneNote, padChord,
//  stab, kick, noiseHit, snare, tom, timpani, riserAt, subDrop)

// ---- FIRE: dispatch one event. dur is converted beats→seconds HERE,
//      via the tempo map, so notes stretch correctly across tempo changes.
function fire(ev,t,t0){const tp=ev[1];
  const durS=(db)=>{const b0=ev[0];return bt(b0+db)-bt(b0);};
  if(tp==='bass')bassNote(t,durS(ev[2]),ev[3],ev[4]);
  else if(tp==='lead')leadNote(t,durS(ev[2]),ev[3]);
  else if(tp==='dr'){/* dispatch on ev[2]: 'k','s','hh',... */}
  // ...
}

// ---- LOOKAHEAD SCHEDULER (mandatory — see mixing.md, failure mode 1)
const LOOKAHEAD=0.6;
function pump(){if(!ctx||ctx.state!=='running')return;const now=ctx.currentTime;
  while(evPtr<events.length&&startT+bt(events[evPtr][0])<now+LOOKAHEAD){
    const ev=events[evPtr],t=startT+bt(ev[0]);
    if(t>now-0.05){try{fire(ev,t,startT);}catch(e){}}   // try/catch: one bad node must not stall the pump
    evPtr++;}}

// ---- AUDIO GRAPH (built on first play; browsers require a user gesture)
function buildGraph(volEl){
  ctx=new (window.AudioContext||window.webkitAudioContext)();
  master=ctx.createGain();master.gain.value=volEl.value/100*0.75;
  const comp=ctx.createDynamicsCompressor();       // LIMITER settings, not glue:
  comp.threshold.value=-9;comp.knee.value=4;comp.ratio.value=5;
  comp.attack.value=0.002;comp.release.value=0.12;
  master.connect(comp).connect(ctx.destination);
  rev=ctx.createConvolver();const rg=ctx.createGain();rg.gain.value=0.55;
  rev.connect(rg).connect(master);
  dly=ctx.createDelay(1);const fb=ctx.createGain();fb.gain.value=0.3;
  const dg=ctx.createGain();dg.gain.value=0.4;
  dly.connect(fb).connect(dly);dly.connect(dg).connect(master);
  const shaper=ctx.createWaveShaper();const curve=new Float32Array(256);
  for(let i=0;i<256;i++){const x=i/128-1;curve[i]=Math.tanh(x*1.8);}
  shaper.curve=curve;drumBus=ctx.createGain();drumBus.gain.value=1.15;
  drumBus.connect(shaper).connect(master);
  mkBufs();startT=ctx.currentTime+0.15;evPtr=0;
  // tempo-synced delay: re-set at each tempo boundary (dotted 8th = 0.75 beat)
  dly.delayTime.setValueAtTime(0.75*60/96,startT);
  dly.delayTime.setValueAtTime(0.75*60/142,startT+bt(32));
  dly.delayTime.setValueAtTime(0.75*60/152,startT+bt(160));
  schedTimer=setInterval(pump,120);pump();}

// ---- TRANSPORT: pause=suspend (clock freezes, scheduler pauses for free)
// play: if no ctx → buildGraph(); else ctx.resume(); playing=true; rAF(loop)
// pause: ctx.suspend(); playing=false
// end/stop: clearInterval(schedTimer); ctx.close(); ctx=null; evPtr=0; redraw

// ---- PIANO ROLL: x maps through the tempo map so time is honest
// const X=b=>bt(b)/TOTAL*w;  py=n=>topPad+(1-(n-lo)/(hi-lo))*(h-pad);
// draw notes as fillRect per event array; drums as short ticks in a bottom
// lane (kick lowest, snare above, hats above that, crash tall); playhead
// vertical line at pos/TOTAL*w; section boundary lines at SECTS beats.
// Colors must survive dark mode: pick mid-ramp hexes and detect
// matchMedia('(prefers-color-scheme: dark)') only for grid/playhead.
```

For the fully expanded, known-good version of every function above, reproduce the patterns from `instruments.md` — they are copy-paste complete.

## 3. Widget vs artifact differences

- Inline visualizer widget: no DOCTYPE/head/body, inline styles, host CSS variables (`var(--border)`, `var(--surface-1)`, `var(--text-secondary)`), Tabler icon font available (`ti ti-player-play`). Transparent outer background.
- HTML artifact: same code works; you may add your own minimal `<style>` and must supply your own icon (a ▶/⏸ text glyph is fine). No storage APIs either way.
- Both: AudioContext may only start after a user gesture — build the graph inside the play-button handler, never on load.

## 4. Extension points (deliberately easy in this structure)

- Mute/solo: give each track family its own bus gain between voice and master.
- Seek: recompute `evPtr` by binary search on `events` for target beat; `ctx.close()` + rebuild, or keep graph and set `startT=now-bt(targetBeat)` before repump.
- MIDI export: the event arrays already are MIDI-shaped (beat, dur, note); serialize to SMF in JS.
- Loop: on end, reset `evPtr=0`, `startT=ctx.currentTime`, keep graph.
