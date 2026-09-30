---
name: product-film
description: Make a product launch / promo video as one self-contained HTML file (picture + synthesized soundtrack driven by a single render(t) function), preview it in the browser, and export a frame-exact 1080p60 MP4 with audio via headless Chrome + ffmpeg. Use this whenever someone wants a launch video, promo or teaser, sizzle reel, animated product demo, "a video of our app/UI", motion graphics, a social clip, or wants to re-time, re-cut, re-score or re-export an existing film.html — even if they never say "HTML".
---

# Product film

A film is **one HTML file**: a 1920×1080 stage, a scene timeline, a pure `render(t)` function that sets every element from film time `t`, and a score built from a precomputed list of sound events. The same file previews live in a browser (scrub, frame-step, play with sound) and exports to MP4 by rendering every frame through headless Chrome and piping it to ffmpeg.

Why this shape: the film stays text (diffable, reviewable, easy to re-time), picture and sound come from one clock so sync is exact, and the export is frame-exact however slow the machine is, because capture never runs in real time.

**Needs:** Chrome or Chromium, `ffmpeg` built with libx264, Node 22+ (the export script uses Node's built-in `WebSocket`).

## Workflow

1. **Storyboard first.** Write a table: scene id, start–end in seconds, the one thing the viewer must understand, and the beat it lands on. Get the user's sign-off on the story before you polish anything. See *Pacing* below; most first cuts are too fast.
2. **Build `film.html`** on the engine contract below. Open it straight from disk (`file://` works). Space plays, ←/→ step one frame, `,` and `.` jump one second, `?t=7.5` opens at a time, `?clean` hides the controls.
3. **Score it.** Put a sound on every on-screen action (see *Sound*).
4. **Verify by looking.** Render stills at key moments and read them (see *Verify*). Check scene boundaries, each click, and each "done" state.
5. **Export** with the capture script. Run it in the background, since it takes about 3–5 seconds of real time per second of film at 60fps. Then pull a few frames from the MP4 itself, and deliver the MP4 with a contact sheet and a line giving its length, resolution and size.

## Engine contract

These rules are what make the export exact. Breaking any one of them shows up as stutter or drift in the MP4 that the live preview hides.

- **`render(t)` is pure.** Every opacity, transform, piece of text and innerHTML is computed from `t`. Don't use CSS `transition`/`animation`, `setTimeout`, `Date.now()` or `Math.random()` inside render, because they run on wall-clock time and capture doesn't. For randomness use a seeded `rng(seed)` or `hash(x)`. Spinners rotate by `t*720deg`, and cursors blink by `Math.floor(t/.5)%2`.
- **Scenes** are `.scene` divs toggled by a timeline map. Inside a scene, work in local time `lt = t - S.x[0]`.
- **Cue times are relative.** Define each scene's cues as an object of offsets (`C1 = {user:.6, click:5.4, done:6.5}`), and derive every absolute time from `S` (`typeSchedule(CMD, S.s2[0]+.7, …)`). Then re-timing one scene moves everything after it automatically. Hardcoded absolute times are how a re-cut breaks. When you do lengthen a scene, grep for leftover absolute times downstream: typing schedules, sound events, the scrub max, `DUR`.
- **Measure positions from the DOM** (cursor targets, anything you point at) with `getBoundingClientRect` divided by the container's current scale. Hardcoded coordinates stop working as soon as the layout shifts.
- **Expose the hook the scripts need:** `window.__film = {render, renderScore, toWav, S, DUR, FPS, ready}`, and set `ready = true` after `document.fonts.ready`. Canvas text never triggers a web-font download, so `document.fonts.load('600 100px Inter')` each face a canvas draws with before you build anything with it.

### Core helpers

```js
const clamp=(v,a=0,b=1)=>Math.min(b,Math.max(a,v)), lerp=(a,b,t)=>a+(b-a)*t;
const win=(t,a,b)=>clamp((t-a)/(b-a));                 // 0→1 progress of t through [a,b]
const expo=t=>t>=1?1:1-Math.pow(2,-10*t), cub=t=>1-Math.pow(1-t,3), smooth=t=>t*t*(3-2*t);
const back=x=>1+2.70158*Math.pow(x-1,3)+1.70158*Math.pow(x-1,2);   // overshoot, for chips/pops
function rng(seed){return()=>{seed|=0;seed=seed+0x6D2B79F5|0;let t=Math.imul(seed^seed>>>15,1|seed);t=t+Math.imul(t^t>>>7,61|t)^t;return((t^t>>>14)>>>0)/4294967296;};}
const hash=x=>{const h=Math.sin(x*127.1)*43758.5453;return h-Math.floor(h);};
const rise=(el,p,blur=10,dy=10)=>{el.style.opacity=p;el.style.filter=p>=1?'none':`blur(${(1-p)*blur}px)`;el.style.transform=`translateY(${(1-p)*dy}px)`;};

// one seeded schedule drives both the visible characters and the key sounds
function typeSchedule(text,a,b,seed){ const r=rng(seed),gaps=[];
  for (const c of text){ let g=.6+r()*.8; if(c===' ')g*=1.5; if('.;,(){}<>/"'.includes(c))g*=1.25; if(c==='\n')g=2.2; gaps.push(g); }
  const sum=gaps.reduce((x,y)=>x+y,0),times=[]; let acc=0; for(const g of gaps){acc+=g;times.push(a+(acc/sum)*(b-a));} return times; }
const typedCount=(times,t)=>{let lo=0,hi=times.length;while(lo<hi){const m=(lo+hi)>>1;if(times[m]<=t)lo=m+1;else hi=m;}return lo;};
```

### Timeline and render skeleton

```js
const S = { s0:[0,3], s1:[3,12], s2:[12,16] }, DUR = 16, FPS = 60, BEAT = .5, BAR = 2;  // 120 BPM
const C1 = { user:.6, tool:1.2, toolDone:2.2, reply:2.5, card:3.9, cursorIn:4.3, click:5.4, done:6.5 };
const REPLY_AT = REPLY.map((_,i)=>C1.reply+i*.12);   // AI text streams word by word
const scenes=[...document.querySelectorAll('.scene')];
const activeScene=t=>{ for (const k in S){ const [a,b]=S[k]; if (t>=a&&t<b) return k; } return Object.keys(S).at(-1); };

function render(t){
  const cur=activeScene(t); scenes.forEach(el=>el.classList.toggle('on', el.id===cur));
  if (cur==='s1'){
    const lt=t-S.s1[0];
    rise($('#ub'), expo(win(lt,C1.user,C1.user+.4)), 6, 14);
    $('#chipi').innerHTML = lt<C1.toolDone ? spinner(lt) : check;          // in-progress → done swap
    replyEls.forEach((e,i)=>rise(e, expo(win(lt,REPLY_AT[i],REPLY_AT[i]+.3)), 10, 6));
    const pressed = lt>=C1.click && lt<C1.click+.12;                        // 120ms press on the button and cursor
    const [tx,ty]=localCenter($('#approve'), $('#win'), 1200), c=cub(win(lt,C1.cursorIn+.1,C1.click-.05));
    cur.style.transform=`translate(${lerp(980,tx,c)}px,${lerp(690,ty,c)}px) scale(${pressed?.86:1})`;
  }
}
function localCenter(el, root, baseWidth){ const r=root.getBoundingClientRect(), e=el.getBoundingClientRect(), k=r.width/baseWidth;
  return [(e.left-r.left)/k+e.width/k*.55, (e.top-r.top)/k+e.height/k*.6]; }
```

Stage fit for preview: `#stage{width:1920px;height:1080px;transform-origin:0 0}` scaled by `Math.min(innerWidth/1920, innerHeight/1080)`. In capture the window is exactly 1920×1080, so the scale is 1.

## Pacing

"It's too fast, nobody can follow it" is the most common note on a first cut.

- Each thing the viewer should notice (a message landing, a tool call finishing, a click, a state change) needs about **0.8–1.2s of dwell** after it lands, before the next thing starts. Hold in-progress states ("Approved, running" with a spinner) for about 1s, or they don't register.
- A UI sequence with about five beats needs **10–12s**, not 4. The assistant-ui film's opener had to be stretched from 4s to 12s after the first review.
- **Stream AI text word by word** (0.12–0.25s between tokens, with a blur-and-rise on each token). Save letter-by-letter typing for humans: commands, composer input.
- **Put big moments on bar lines.** At 120 BPM a bar is 2s, so scene cuts, camera moves, drops and crashes land on even seconds.
- Keep a slow camera drift (for example, scale 1→1.035 across a held shot) so static holds don't look frozen.

### Motion recipes that read well

- **Pull-back reveal:** start zoomed in on one line of UI, then pull back to the full app on the downbeat. To zoom by `SM` so that stage point `p0` ends up at screen point `T0`, set the transform origin to `F = (SM*p0 - T0)/(SM-1)` and use `scale(Math.pow(SM, 1-expo(progress)))`.
- **Chat that grows:** put messages in an inner column and `translateY(-scroll)` it as new content arrives, eased with `expo` over about 0.5s. The cursor lives outside the scrolling column, so add the scroll back into its target y.
- **Suggestion chips:** pop in with a 0.15s stagger using `back()` scale 0.85→1. On the click, fade the chip row out and slide a user bubble with the chip's text in.
- **Result cards:** slide up 16px while fading in. After that, animate one detail at a time (a toggle flips, days fill in one by one, a success line appears).

## Sound

The score is synthesized with WebAudio from a precomputed list `EVENTS = [[filmTime, fn(audioTime)], …]`. Live preview schedules the events about 120ms ahead on the audio clock, and the picture follows the audio clock. Export renders the **same list** through an `OfflineAudioContext` to a WAV file, so sync is exact.

```js
const EVENTS=[]; const at=(t,fn)=>EVENTS.push([t,fn]);
// instruments (all take audioTime): kick, clap, hat, bass(f), pad(notes,len), pluck(f), blip(f), key(big?), chord, crash, riser(dur), sub
for (let b=0; b*BEAT<DUR; b++){ const t=b*BEAT; if (drumsOn(t)){ at(t,a=>I.kick(a)); if(b%2) at(t,a=>I.clap(a)); at(t+BEAT/2,a=>I.hat(a)); } }
const o=S.s1[0];
REPLY_AT.forEach((r,i)=>at(o+r,a=>I.blip(a,2300+i*80)));   // soft blip per streamed token
at(o+C1.click,a=>I.key(a)); at(o+C1.done,a=>I.chord(a));   // click → key tick; completion → rising 3-note pluck
CMD_T.forEach(ct=>at(ct,a=>I.key(a)));                      // typing sounds come from the same schedule as the text
at(S.s1[1]-1,a=>I.riser(a,1)); at(S.s2[0],a=>{I.crash(a);I.sub(a);});   // build into each scene cut
EVENTS.sort((x,y)=>x[0]-y[0]);

async function renderScore(){                 // swap the live context for an offline one, replay every event
  const sr=48000, oac=new OfflineAudioContext(2,Math.ceil(sr*DUR),sr), saved=[AC,OUT_BUS,NOISE];
  AC=oac; buildBus(oac);                      // compressor → master gain; seeded noise buffer
  try { EVENTS.forEach(([ft,fn])=>fn(ft)); return await oac.startRendering(); } finally { [AC,OUT_BUS,NOISE]=saved; }
}
```

`toWav(audioBuffer)` writes a standard 16-bit PCM RIFF header plus interleaved samples. Build the noise buffer from `rng(seed)`, and derive noise start offsets from `hash(at)`, so the WAV comes out identical on every render.

A simple chord loop sounds fine: Am, F, C, G, two bars each. Drop the drums for the second before a cut so the riser has room. Check the levels: the peak must stay below 1.0 (no clipping), and the body of the film should sit around -20 to -26 dBFS RMS per scene. The intro and outro can be quieter.

## Verify

Render stills and look at them yourself before you export. Most bugs are only visible in a frame: text clipped by its container, overlapping cards, an element that scrolled off, the cursor missing its button at click time, a scene boundary that flashes.

```bash
CH="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"   # or google-chrome / chromium
for t in 1.5 5.2 8.0 9.8 14.8; do
  "$CH" --headless=new --disable-gpu --hide-scrollbars --window-size=1920,1080 --virtual-time-budget=4000 \
    --screenshot="frames/$t.png" "file://$PWD/film.html?t=$t&clean"
done
```

Pick times just after each cue lands, just after each click, and a few frames either side of every scene boundary. For a contact sheet, make a small HTML grid of those PNGs with a timestamp on each, and screenshot it the same way.

Also check that `render(t)` stays fast (well under 16ms, or live preview stutters) and that the page throws no errors. A quick loop inside the page's console that times `__film.render(t)` across each scene covers both.

## Export

Save this as `film-capture.mjs` next to the film, and run it in the background:

```bash
node film-capture.mjs film.html film.mp4          # 60fps, whole film
node film-capture.mjs film.html test.mp4 30 3 12  # 30fps, only 3s–12s (quick check of one scene)
```

```js
// node film-capture.mjs <film.html|url> <out.mp4> [fps=60] [from=0] [to=DUR]
import { spawn } from 'node:child_process';
import { mkdtempSync, rmSync, readFileSync, writeFileSync, existsSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join, resolve } from 'node:path';
import { pathToFileURL } from 'node:url';

const [target, out, fpsArg, fromArg, toArg] = process.argv.slice(2);
if (!target || !out) { console.error('usage: node film-capture.mjs <film.html|url> <out.mp4> [fps] [from] [to]'); process.exit(1); }
const FPS = +fpsArg || 60;
const CHROME = process.env.CHROME || ['/Applications/Google Chrome.app/Contents/MacOS/Google Chrome',
  '/usr/bin/google-chrome', '/usr/bin/google-chrome-stable', '/usr/bin/chromium', '/usr/bin/chromium-browser'].find(existsSync);
const url = /^(https?|file):/.test(target) ? target : pathToFileURL(resolve(target)).href;
const sleep = ms => new Promise(r => setTimeout(r, ms));

// private profile + port 0: never collides with a leftover Chrome from an interrupted run
const dir = mkdtempSync(join(tmpdir(), 'film-chrome-'));
const chrome = spawn(CHROME, ['--headless=new', '--remote-debugging-port=0', '--window-size=1920,1080', '--hide-scrollbars',
  '--autoplay-policy=no-user-gesture-required', '--disk-cache-size=1', '--no-first-run', `--user-data-dir=${dir}`, 'about:blank'], { stdio: 'ignore' });
const cleanup = () => { try { chrome.kill('SIGKILL'); } catch {} rmSync(dir, { recursive: true, force: true }); };
process.on('exit', cleanup); process.on('SIGINT', () => process.exit(130));

let port; for (let i = 0; i < 100 && !port; i++) { try { port = readFileSync(join(dir, 'DevToolsActivePort'), 'utf8').split('\n')[0]; } catch { await sleep(100); } }
const targets = await (await fetch(`http://127.0.0.1:${port}/json`)).json();
const ws = new WebSocket(targets.find(t => t.type === 'page').webSocketDebuggerUrl);
await new Promise(r => (ws.onopen = r));
let id = 0; const pending = new Map();
ws.onmessage = e => { const m = JSON.parse(e.data); if (m.id && pending.has(m.id)) { pending.get(m.id)(m); pending.delete(m.id); } };
const send = (method, params = {}) => new Promise(r => { const i = ++id; pending.set(i, r); ws.send(JSON.stringify({ id: i, method, params })); });
const ev = async expr => { const r = await send('Runtime.evaluate', { expression: expr, awaitPromise: true, returnByValue: true });
  if (r.result.exceptionDetails) throw new Error(r.result.exceptionDetails.exception?.description || r.result.exceptionDetails.text); return r.result.result.value; };

await send('Emulation.setDeviceMetricsOverride', { width: 1920, height: 1080, deviceScaleFactor: 1, mobile: false });
await send('Page.navigate', { url: url + (url.includes('?') ? '&' : '?') + 't=0&clean' });
for (let i = 0; i < 200 && !(await ev('!!window.__film?.ready').catch(() => false)); i++) await sleep(100);
const DUR = await ev('__film.DUR'), from = +fromArg || 0, to = +toArg || DUR;

const wav = join(dir, 'score.wav');
const b64 = await ev(`(async()=>{const u=new Uint8Array(await __film.toWav(await __film.renderScore()).arrayBuffer());let s='';for(let i=0;i<u.length;i+=0x8000)s+=String.fromCharCode(...u.subarray(i,i+0x8000));return btoa(s);})()`);
writeFileSync(wav, Buffer.from(b64, 'base64'));

const ff = spawn('ffmpeg', ['-y', '-loglevel', 'error', '-f', 'image2pipe', '-framerate', String(FPS), '-c:v', 'mjpeg', '-i', '-',
  '-ss', String(from), '-i', wav, '-map', '0:v', '-map', '1:a', '-c:v', 'libx264', '-preset', 'slow', '-crf', '16', '-pix_fmt', 'yuv420p',
  '-c:a', 'aac', '-b:a', '256k', '-shortest', '-movflags', '+faststart', resolve(out)], { stdio: ['pipe', 'inherit', 'inherit'] });
const ffDone = new Promise(r => ff.on('close', r));
const N = Math.round((to - from) * FPS), t0 = Date.now();
for (let f = 0; f < N; f++) {
  await ev(`__film.render(${(from + f / FPS).toFixed(6)})`);
  const shot = await send('Page.captureScreenshot', { format: 'jpeg', quality: 95, fromSurface: true });
  if (!ff.stdin.write(Buffer.from(shot.result.data, 'base64'))) await new Promise(r => ff.stdin.once('drain', r));
  if (f % (FPS * 4) === 0) console.log(`frame ${f}/${N} · ${((Date.now() - t0) / 1000).toFixed(0)}s`);
}
ff.stdin.end();
const code = await ffDone; ws.close();
console.log(code === 0 ? `wrote ${out} (${N} frames, ${(to - from).toFixed(2)}s @ ${FPS}fps)` : `ffmpeg failed (${code})`);
process.exit(code);
```

The script renders the score to a WAV file in a private temporary Chrome profile, then drives `__film.render(frame/FPS)` and screenshots every frame into ffmpeg (H.264, CRF 16, yuv420p, AAC 256k, `+faststart` so the file streams). Afterwards it deletes the profile. Frames go through a pipe, so no PNGs are written to disk. You only need free space for the MP4 itself (about 15MB for a 1-minute film).

After export, pull a few frames from the MP4 with `ffmpeg -ss <t> -i film.mp4 -frames:v 1 check.jpg` and look at them. Confirm with `ffprobe` that the duration matches `DUR` and that there's an audio stream.

## Troubleshooting

- **Motion stutters in the MP4 but is smooth in the browser:** something is reading wall-clock time (a CSS transition or animation, `Date.now`, `setTimeout`). Move it into `render(t)`.
- **Export hangs before the first frame:** `window.__film.ready` never became true. The usual causes are a script error (open the file in a browser and check the console) or a missing `ready = true` after fonts load.
- **Wrong or fallback fonts:** Google Fonts needs network access. For offline exports, self-host the font files with `@font-face`.
- **Orphaned Chrome processes after an interrupted run:** `pkill -f film-chrome-`. The script uses its own profile and a random port, so orphans can't collide with the next run, but they do use memory.
- **Picture and sound drift in the browser preview:** preview only. Picture follows the audio clock once audio is running, and the export is always exact.
