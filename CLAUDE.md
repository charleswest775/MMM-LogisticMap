# MMM-LogisticMap — context for Claude sessions

Charles's MagicMirror² module: the logistic map's road to chaos, for his hallway mirror, as one
page in a rotation of pages. Split out of MMM-ChaosTheory on 2026-09-28 with its history (it was
the `logistic` simulation of that module's chaos page).

## Files

- `MMM-LogisticMap.js` — module shell: one canvas plus an HTML caption (equations + live
  readout, updated 2×/s). Starts again every `cycleSeconds` and on each `resume()`. Loop:
  `setTimeout` until a frame is due, then one `requestAnimationFrame`. `suspend()` stops it; a
  sim with `resting = true` is polled only every 500 ms (this one never rests); while
  MagicMirror fades the module out (`hidden` is set at the start, `suspend()` comes after),
  frames draw nothing. `turns: { of, at }`: only every nth showing; otherwise the wrapper gets
  `display: none` and nothing starts.
- `simulations/logistic.js` — the `logistic` sim on `window.LogisticSimulations`, plus
  `period(r)` and `lyapunov(r)`. The bifurcation diagram (r from 2.8 to 4) is painted column by
  column into an ImageData over `paintSeconds` (12), only the new columns put each frame; then
  a cobweb sweeps r back and forth over `sweepSeconds` (40) each way, redrawing only its own
  square each frame, and the marker on the diagram moves only twice a second (so frames don't
  touch both). `paintSeconds` and `sweepSeconds` come from the module config but aren't in its
  `defaults`. A page of about a minute shows the paint and one sweep.
- UMD-style, so the maths runs in Node: `tests/logistic.test.js` (`node --test`, no
  dependencies): periods 1, 2, 4, 8, chaos; the period-3 window at 1 + √8; the first
  bifurcation points and Feigenbaum ratio; the Lyapunov exponent (ln 2 at r = 4).
- `node_helper.js` — the stats panel (`statsPanel: true`): CPU of Electron and cage, per core,
  temperature, from `/proc`, only while shown.
- `dev/preview.html` — runs the module in a desktop browser (`python3 -m http.server` in the repo,
  then `/dev/preview.html?paintSeconds=4&sweepSeconds=20`).

MMM-ChaosTheory has the same simulation (`logistic`): a fix to it probably belongs in both.

The shell (`MMM-LogisticMap.js`, `node_helper.js`'s stats panel, `dev/preview.html`) is shared
in spirit with the sibling modules (MMM-ChaosTheory, the other split-out chaos modules, and
MMM-FractalZoom, MMM-Atom, MMM-Chladni, MMM-SacredGeometry, MMM-Tilings, MMM-PlanetsDance,
MMM-SnowCrystal, MMM-NightSky, MMM-PhotoDeck, MMM-StandardMap, MMM-ChaoticWaterwheel, MMM-DoubleSlit, MMM-Sandpile, MMM-Harmonograph): a fix there probably belongs in the siblings too.

## Measured cost on the Pi

900², 20 fps, over 60 s, Electron + cage: 73% of a core, at 17 fps. Hidden: 0.3% (baseline
0.2%). It never rests while shown: the cobweb keeps moving.

## Performance findings on the Pi (measured)

- A frame that changes the canvas costs ~2%/fps fixed; beyond that, cost scales with the
  **bounding box of everything changed in the frame**. Full redraws of a 900² canvas at 20 fps
  saturate the pipeline (~150%). JS is never the bottleneck (<3 ms/frame).
- So: draw incrementally, keep each frame's changes spatially compact, and rest when the
  picture is static. Line width, opacity, `rAF` vs timer made no difference.
- MagicMirror applies `electronSwitches` after app ready, so `remote-debugging-port` can't be set
  that way; use `debugStats: true` and a `grim` screenshot to see fps on the Pi.

## Hard constraints: the target device

- **Raspberry Pi 3 B+, 905 MB RAM, 64-bit Debian 13.** Mirror runs Electron 42 in a cage
  Wayland kiosk.
- **No GPU acceleration, and it can't be enabled**: the Pi 3's VideoCore IV only does GLES 2.0,
  Chromium needs ES 3.0 (tested). All canvas drawing is CPU. **No WebGL / three.js.**
- Screen will be **portrait 1200×1920** once mounted (Dell U2413, rotated). Design for portrait.
- Electron baseline is ~0.5% of one core. **Measure, don't guess**: on the Pi,
  `~/.cache/mm-sample.sh 60` prints Electron CPU% and RSS over 60 s. Record before/after numbers
  in the README.
- The mirror rotates pages every 15-30 s (MMM-pages, which hides/shows modules). `suspend()` and
  `resume()` must fire on page changes, or the loop burns CPU 24/7.

## Deploying and testing

- This repo is public so the Pi can `git clone`/`git pull` without credentials.
- Pi access: `ssh fatherson@raspberrypi.local` (key auth). Module path:
  `~/MagicMirror/modules/MMM-LogisticMap`. Restart: `pm2 restart MagicMirror`
  (pm2 is in `~/.npm-global/bin`). Logs: `pm2 logs MagicMirror`.
- The mirror's **config.js lives in a separate private repo**, `charleswest775/magicmirror-setup`
  (cloned at `~/dev/magicmirror-setup`). Add the module's config block there, then
  `./deploy.sh diff` and `./deploy.sh push` (push validates config before restarting).
  Don't hand-edit config.js on the Pi without `./deploy.sh pull` afterwards.
- Faster iteration: run it in a desktop browser (`dev/preview.html`), then confirm performance
  on the Pi.
- Commit as Charles's GitHub noreply address (set in this repo's git config).
