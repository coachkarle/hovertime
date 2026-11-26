# HoverTime handoff: reusing the code and music

For people and AI assistants (Claude or otherwise) building another tool that wants to borrow HoverTime's timer, music engine, Float window, or card system. Read this before copying code out of `index.html`.

> **License: none yet.** The repository has no LICENSE file, so by default its owner keeps all rights. Ask the owner, Karle Delo ([@coachkarle](https://linktr.ee/coachkarle)), before reusing code or music in another project, and ask them to add a license if reuse is welcome.

## The shape of it

- **One file, no build, no dependencies.** Everything lives in `index.html` (about 7,900 lines, 580 KB): HTML, CSS, and one script. All sound is synthesized live with the Web Audio API. There are no audio files.
- **Five modules**, each an IIFE that returns a small API. All of them are exposed on `window.PocketTimer` for tests and reuse:

| Module | Global | What it is | Reuse difficulty |
|---|---|---|---|
| 1. Timer core | `TimerCore`, `Rotation`, `PartnerPrompts` | Countdown state machine with no DOM or audio; equal-turn splitting; partner prompts | Copy as is |
| 2. Synth library | `SynthLib` | Instrument voices and effects built on Web Audio | Copy as is |
| 3. Music tracks | `Tracks` | 14 songs written as code (data plus a step function) | Copy with modules 2 and 4 |
| 4. Audio engine | `AudioEngine` | Master chain, lookahead player, chimes, ducking | Copy with modules 2 and 3 |
| 5. Themes and UI | `Themes`, `UI` | 10 visual vibes, renderers, cards, Float, Full screen | Borrow patterns, not code |

Find a module by searching for `MODULE 1 of 5` through `MODULE 5 of 5`.

## Recipe 1: the timer core alone

`TimerCore` has no DOM or audio dependencies, so you can drop it into any page.

```js
const timer = TimerCore.create();          // optional: create(() => performance.now())
timer.on(reason => render(reason));       // 'load' | 'start' | 'pause' | 'reset' | 'adjust' | 'finish'
timer.load(5 * 60000);                     // ms
timer.start(); timer.pause(); timer.toggle(); timer.reset(); timer.adjust(60000);
setInterval(() => timer.tick(), 25);       // call often; it emits 'finish' at zero
timer.remaining(); timer.total; timer.loaded; timer.running; timer.finished; timer.endAt;
TimerCore.parse('2:30');                   // → 150000  (accepts 5, 5:00, 90s)
TimerCore.format(150000);                  // → '2:30'
```

**Background tabs throttle timers.** HoverTime ticks from a tiny Web Worker (search `background-safe ticker`) so the countdown and chimes stay accurate when the tab is hidden. Copy that pattern too.

## Recipe 2: play the music in another app

You need modules 2, 3 and 4. Browsers block audio until the user clicks or presses a key, so unlock it from a user gesture:

```js
button.onclick = async () => {
  await AudioEngine.unlock();                                   // resumes the AudioContext
  AudioEngine.setVolume(0.75);
  AudioEngine.setTrack('orbit', /*running*/ true, /*endAt*/ performance.now() + 10 * 60000);
};
// stop: AudioEngine.setTrack(null, false, 0)
// list: AudioEngine.Tracks.list.map(t => [t.id, t.name])
```

**Track IDs, grouped by set:**
- **calm:** `lofi`, `offbeat`, `airy`, `upbeat`
- **chip** (8-bit): `arcade`, `quest`, `bounce`
- **synth:** `nightdrive`, `neonrun`, `horizon`
- **show** (Game Show): `spotlight`, `lightning`
- **ambient** (Sci-Fi): `deepspace`, `orbit`

Tracks marked `tempoFollow` step up their tempo near the end of a timer.

The engine also schedules the **end-of-timer chimes** from the timer's end time. Call `AudioEngine.onTimer({ running, endAt, finished, rot })` whenever the timer changes. `rot` takes Rotation switch times for turn-change chimes; pass `null` if you don't use turns. You also need to call `AudioEngine.tick(document.hidden, endAt)` regularly from your ticker.

### Render a track offline (no speakers, no user gesture)

This is how loudness was measured, and it's the easiest way to test a track in isolation:

```js
const ctx = new OfflineAudioContext(2, 22050 * 60, 22050);
const L = SynthLib.create(ctx), tr = Tracks.byId.lofi, out = ctx.createGain(); out.connect(ctx.destination);
const song = tr.create(ctx, out, AudioEngine.mulberry32(11), L, 0);   // seeded RNG → repeatable
const s16 = 60 / tr.bpm / 4;
for (let i = 0; 0.1 + i * s16 < 59; i++) song.step(i, 0.1 + i * s16);
const buffer = await ctx.startRendering();                           // measure RMS, export WAV, etc.
```

## Recipe 3: write a new track

A track is `{ id, name, bpm, tonic, trim, set?, tempoFollow?, create(ctx, out, R, L, startStep) }`. `create` returns `{ step(i, t), dispose(t), setTempo?(mult) }`.

- **`step(i, t)`** is called once per 16th note on an un-swung grid. `t` is the AudioContext time, scheduled ahead, so schedule notes at `t`; never use `currentTime` for timing. `bar = i >> 4`, `s = i % 16`. The songbook engine also supports 12 steps per bar (Quest).
- **`R`** is a seeded RNG. Use it, not `Math.random`, for musical choices, so a song is repeatable from its seed.
- **`L`** is the synth library.
  - **Instruments:** `rhodes`, `felt`, `wurli`, `mallet(kind: 'vibe'|'bell'|'glass')`, `pad`, `bass`, `kick`, `snare`, `clap`, `hat`, `rim`, `shaker`.
  - **Building blocks:** `osc`, `gain`, `filter`, `noise(kind, seconds)`, `chain(...nodes)`, `mtof`.
  - **`L.scope()`:** gives managed buses and effects: `gain`, `filter`, `reverb`, `chorus`, `tapeWow`, `tremolo`, `autopan`, `drive`, `lfo`, `hiss`, `crackle`, `stop`, `disconnect`. Build all long-lived nodes through the scope, so `dispose(t)` can stop and free everything with `S.stop(t)` then `S.disconnect()`.
- **Shared helpers:** search `const helpers = R =>` for `rr`, `chance`, `pick`, `weighted`, `voiceLead`, `fit(scale, chord)` (drops scale notes that rub a half step against the chord), and `phrase(scale, len, start, history)` (a stepwise melodic phrase that avoids repeating recent shapes).
- **Data-driven engines:** `makeChipTrack`, `makeSynthTrack`, `makeShowTrack` and `makeAmbientTrack` turn song data (chords, 8-bar tunes, section recipes) into tracks. Adding a song to an existing style is mostly writing data.
- **Register it** in the `list` at the end of module 3.

### What made the music better (the audits)

Each song was audited for being "sleepy and repetitive" and fixed with small, cheap changes. **No new synthesis, no extra CPU on Chromebooks.** Reuse these moves:

1. **Sections every 8 bars.** An 8-bar intro, then layers come and go: melody on or off, a shaker or light 16th hats in some sections, a breakdown now and then.
2. **Call and answer melodies.** On even bars, a 4–5 note phrase from `H.phrase` over `H.fit(scale, chord)`. On odd bars, a 2–3 note answer that lands on a chord tone.
3. **Returning tunes get a twist.** Count how often each section has played. On a repeat, sometimes lift the second half an octave, end a long note on a neighboring chord tone, or change the chip pulse width.
4. **Transitions.**
   - A snare fill rising across the last 4 sixteenths of a section.
   - A soft crash when a new section starts.
   - A filtered-noise riser out of quiet sections. Search `function riser`.
5. **Steady over random.** Random drum dropouts sound like glitches. Use a quieter, steady pattern instead.
6. **A modest tempo lift** for sleepy songs: 6–8%.

**Keep loudness matched.** All songs sit near **-18.7 dB RMS** after their `trim`, measured over about 110 s and two seeds with the offline render above. Re-measure after any change and adjust `trim`. The ambient songs vary about ±0.7 dB between runs because of random noise layers, so don't chase small differences there.

### Chimes and ducking

- **Routing:** cues go to `chain.cueIn` (gain 1.6), music to `chain.musicIn`. Both pass through a high-pass filter, a limiter, a soft clipper and the volume control.
- **Ducking:** a `chain.duck` gain dips the music about 8 dB around the 1-minute chime and each turn switch. Search `music dips about 8 dB`.
- **Cue styles per vibe:** `default`, `chip`, `show` and `hud`, set with `AudioEngine.setCueStyle()`.

## Recipe 4: float over other apps (Document Picture-in-Picture)

"Float" moves the live timer panel into an always-on-top window that stays over full-screen slides. Desktop Chrome and Edge 116+ only. Search `Document Picture-in-Picture`.

**Pattern:**
1. `documentPictureInPicture.requestWindow({ width, height })` from a click.
2. Clone every `<style>` and stylesheet `<link>` into the new document.
3. Move the existing panel element in with `appendChild`. Don't rebuild it. The same nodes, listeners and state carry over.
4. Add keyboard and pointer listeners to the new document too. Run the animation loop on `pipWin.requestAnimationFrame`.
5. On `pagehide`, move the panel back with `insertBefore`.

**Gotchas we hit:**
- **Element lookups break in Float.** After the move, `document.getElementById` can't find anything inside the panel, because it lives in the Float window's document. Grab element references once at startup (the `el.*` object) and use those. One `$('id')` lookup at click time broke "New note" in Float.
- **Create nodes in the right document.** Use `someEl.ownerDocument.createElement`, and `ownerDocument.defaultView` for computed styles.
- **Resizing can fail.** `pipWin.resizeTo` needs a user gesture and can be refused, so layouts must fit whatever size the window has.
- **Auto-Float exists.** Chrome can float automatically when the tab is hidden while media plays: `mediaSession.setActionHandler('enterpictureinpicture', …)`.

## Recipe 5: cards (directions and protocols)

- **Data shape:** `{ id, title, min, text, kind? }`.
  - `text` is plain text, one line per step.
  - `min` loads into the timer (0 means none).
  - `kind: 'break'` shows "Back at 10:42", computed as `Date.now() + remaining`, then "Welcome back".
- **Storage:** built-ins live in `PROTOCOLS`. The user's edits and own notes live only in `localStorage['ht-cards']` as `{ edits: {id: card}, custom: [card] }`.
- **Rendering** (search `function styled`):
  - Lines starting `1.`, `-` or `•` become lists with hanging indents.
  - A line ending in `:` becomes an italic lead-in.
  - A short `Label:` becomes bold, and so does a leading count ("**3** key takeaways").
  - `**word**` becomes bold.
  - Everything is built from text nodes, so nothing typed can become markup. Keep it that way.
- **Fit to space:** in Float, Full screen and wide layouts, a binary search finds the largest `--cf` font size that fits (search `function fitCard`). Refit after web fonts load (`document.fonts`), or text overflows.
- **Analytics:** `htCount('card/<id>')` sends the protocol name only, never typed text.

## Recipe 6: a vibe (theme)

- **Definition** (search `const list = [` in module 5): `{ id, label, colors | palettes, font, music: { set, first }, tempo, cues, banners, confetti, renderer }`.
- **Renderer interface:** `{ draw(view), drawMini(view), invalidate(), onAdjust?(), onFinish?() }`.
  - `view` is one shared timeline: `rem`, `total`, `f`, `share(edge)` (how much of the picture has drained, with turn switches), `fin`, `finished`, `elapsed`, `win`.
  - Renderers redraw only when a cache key changes.
- **Colors:** each theme derives its color tokens so every text pair passes WCAG AA. Search `Color.pick`.

## Hard-won gotchas (layout)

- **Page zoom changes measurements.** The page size grip uses CSS `zoom`. `getBoundingClientRect()` then includes the zoom, but `clientWidth` and canvas drawing coordinates don't. Divide measurements by `el.currentCSSZoom` before drawing (Sunrise and Synthwave drew their digit box in the wrong place until this was fixed), and multiply canvas resolution by it so drawings stay sharp.
- **Digits can overflow tight layouts.** A narrow Full screen side, a small Float window, or 9:59 → 10:00 can make them too wide. `fitClock()` shrinks the font with `!important`, and runs on layout changes only, not every frame.
- **Centering a column can push its top off-screen** when content is taller than the container. Use `justify-content: safe center`.
- **You can't nest `:has()` inside `:has()`.** Set a class from JavaScript instead.
- **Rebuilding a node breaks double-click.** If the first click replaces the element, the browser never fires `dblclick`. Change classes on the existing elements instead.
- **A plain `<select>` is as wide as its longest option.** `field-sizing: content` sizes it to what's shown.

## Testing

These scripts were run during development but are **not in the repo**. Ask if you want them added.

- **Playwright, headless Chromium**, for click-throughs of every feature in all 10 vibes, plus contact-sheet screenshots of each vibe.
- **A real Float window works headless.** `ctx.waitForEvent('page')` after clicking Float returns the Document Picture-in-Picture window, and you can drive it like any page. Its default size is the page size, so small-window layouts still need a manual check.
- **Loudness:** the offline render above, RMS per track, compared before and after any change.

## Privacy

Nothing leaves the browser except an anonymous, cookie-free GoatCounter visit count with event names like `start/<vibe>`, `float`, `card/<id>` and `music/<id>`. Settings, cards and the Full screen divider and page size settings stay in `localStorage`. Keep it that way if you borrow the analytics hook (`window.htCount`).
