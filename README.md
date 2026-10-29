# ApexRL - watch an AI learn to drive the Nürburgring Nordschleife, in day, night and rain

One car, top-down view, real reinforcement learning. The car starts knowing **nothing** and learns from its **9 bumper sensors**, speed, steering, slip and how wet/dark it is. No map, no position, no track data.
The track is the real Nordschleife (about 20.7 km) built from a GPS trace. Learning algorithm: **PPO**, written from scratch in JavaScript.

## NEW: the real 3D Nürburgring model (track "3D model: real walls", default)

`assets/nurburgring.glb` (Sketchfab "Nürburgring racing circuit (FR Legends mod)" by Unprofessional Modeler, CC-BY-4.0: credit the author if you publish) is used in two ways:

1. **Drawn in 3D** with three.js (press **V**): real terrain, trees, road, rails, with your car sprite, sensor rays, night and rain.
2. **Turned into the driving world.** `tools/extract_glb.py` (needs numpy, scipy, scikit-image; already run, output is `web/data/nord_glb.js`) snaps a centreline onto the model's road and measures road width, elevation and where the walls are. The simulator raycasts against that 2D corridor (asphalt, then a grass strip, then the wall), not against the 3D mesh, so training stays fast. PPO, observations (14 inputs) and actions are unchanged.

Facts and limits:
- The model is in metres; the lap measures about 20.8 km and the road is about 7.8 m wide.
- The model has walls on only about half of each side. Where it has none, a 4 m grass strip with a hard edge is used instead.
- This track uses a true-size car (4.6 x 1.9 m) with speed-sensitive steering (`PHYS.real` in `sim.js`). The built-in teacher laps it in about 8:45. Brains trained on the old tracks do not transfer.
- Height is visual only; the physics is flat. Ghosts are drawn only in the top-down view.
- The model's road and rail textures break in its merged atlas, so they are replaced by generated ones.
- Headless test: `node tools/headless_test.js 2 glb imitate`.

## NEW: drive it yourself, then let the AI learn from you

Press **D** (or "Drive yourself") to take the wheel. **Up** = throttle, **Down** = brake, **Left/Right** = steer, **Space** = handbrake, **R** = back to the start line, **P** = pause (Space no longer pauses while you drive). Simulation speed is forced to 1x while you drive.

- **Recording.** While you drive, every physics step stores what the AI would have seen (its 14 sensor numbers) and what you did with wheel and pedals. The last 0.8 s before each crash is thrown away (it is the mistake, not the lesson). **Discard last run** removes a whole run. The run in progress is kept when you stop driving, restart, save or train.
- **After a crash** you carry on from about 40 m before it, so every part of the 20 km gets covered without repeating the start. Tick "restart at the start line" for full laps; a run only counts as a lap when it starts at the line.
- **Save my laps / Load saved laps** append to / read `saves/demos.json` (about 1.5 MB per 10 minutes of driving), so several sessions add up. Delete the file to start over.
- **Learn from my laps** trains the AI to copy your steering and pedals (behaviour cloning, like "Study this driver" but with *you* as the teacher), runs a noise-free test drive, then normal reinforcement learning continues. **Keep copying my laps while reinforcement learning runs** (on by default) adds two small copy steps after each PPO update, so RL refines your driving instead of wandering into wrong paths.
- Recordings from the 3D-model track and from the arcade tracks are kept apart (different car physics); only matching ones are used.

Honest limits (measured headless with a scripted "human" pressing the same keys): copying alone gets the car a few km round the 20.8 km lap (about 5-9% in the test drive), about the same as the built-in teacher imitation. More driving does not fix this alone, because a copy drifts into situations it never saw. Treat it as a head start that RL builds on, and drive the hard corners several times. In a short test I could not show the copy-anchor beating plain RL (8% vs 6% after 3 minutes, within noise).
The AI has only steering and throttle/brake outputs, so your handbrake is learned as "brake hard". The handbrake physics (extra drag, nose swings round, the car slides on) is an approximation.
Headless check: `node tools/headless_demo_test.js 240 glb 0.6`.

## Quick start (Windows)

1. Install **Python 3.8+** from python.org (tick "Add python.exe to PATH"). Nothing else.
2. Unzip, double-click **`start.bat`**. Browser opens at http://127.0.0.1:8765.

No `pip install`, no internet, **no API key needed**.

## Your car images (assets folder)

| File | Used when |
|---|---|
| `assets/car.png` | day and dry |
| `assets/car_lights.png` | night or rain (headlights on). Fades in smoothly |

Replace both with your own PNGs (same car, same orientation, ideally the same canvas size). `car_lights.png` that ships in the zip is only a **placeholder** (the day car with glowing lights); overwrite it with yours.
A white background is removed automatically. If `car_lights.png` is missing, the day car is used with simple glowing headlights.
Settings at the top of `web/js/render.js`: `CAR_IMG.facing` (which way the nose points in your PNG: left/right/up/down; the BMW faces `left`) and `CAR_IMG.length` (size). Ctrl+F5 after changing images.

## Day / night / rain

Controls (Tab) -> **Conditions**:

- **Time of day:** Day, Night, or Auto cycle (24 h = 12 simulated minutes, with dusk and dawn).
- **Weather:** Sunny, Rain, or Random showers. The track dries slowly after rain.
- **Make conditions harder as it improves:** a curriculum: day -> night -> rain -> night + rain. It moves on when the last 10 attempts average at least 20% of the lap (and at least 30 attempts at that stage).

What changes:

| | Night | Rain |
|---|---|---|
| Look | dark overlay, lit lamp posts, headlight cone | rain streaks, grey overcast, spray behind the car |
| Car sprite | lights version | lights version |
| Sensors | range cut by about 40%, a little noise | range cut by about 20%, more noise |
| Physics | none | grip about 40% lower: it slides (slip meter), brakes and accelerates worse, steering reacts later |
| What the AI knows | a "night" input | a "wetness" input and a slip input |

The table in Controls shows best/average distance **per condition**, so you can see how rain and night hurt it (and later how training recovers it).

## Learning from real drivers

Controls -> **Learn from real drivers**.

- **Reference lap times** (real, published): Timo Bernhard 5:19.546 (Porsche 919 Hybrid Evo, 2018), VW ID.R 6:05.33, Stefan Bellof 6:11.13 (Porsche 956, 1983), Maro Engel 6:29.09 (Mercedes-AMG One, 2024), Porsche 911 GT2 RS (Manthey) 6:43.30, Vincent Radermecker 7:04.957 (Xiaomi SU7 Ultra, 2025), Tom Schwister 7:25.23 (Tesla Model S Plaid), Honda Civic Type R 7:44.88. Sources are listed in `web/js/drivers.js`. Records get broken, and the official lap is ~20.8 km with a flying start, so check them if you publish anything.
- **Ghost:** a translucent car that drives the lap in the driver's **real time**. The HUD shows your gap ("+1.3 s behind Vincent"). Or choose **My AI's best lap** to race your own best lap.
- **Study this driver (imitation):** the AI first copies a **teacher driver**, then normal RL continues from that head start. The teacher is paced like the chosen driver (slower drivers = a more cautious teacher). After studying, the actor is frozen for a few updates so the value estimate can catch up.

**Be aware:** I had no access to real telemetry (racing lines, speeds), only lap times. So the teacher is a synthetic driver built from the track layout (it uses the layout, which the learner never sees). The ghost's speed profile uses the same shape scaled to the real lap time, so its *total time* is exact but its speed through each corner is an estimate. The game car is also much quicker than real cars (the teacher laps in about 4:40 dry), so beating a ghost is not a real-world comparison. If you ever get real telemetry as CSV, it can be plugged into `Expert`/`buildGhostClock` in `web/js/drivers.js`.

In my headless tests, imitation reached several km (even the full lap distance in places) within seconds of starting RL, versus minutes from scratch. After that, RL can wobble and drop for a while before it settles, which is normal. Rain/night are harder to imitate and to learn.

## 3D chase view (new)

Press **V** (or the "View" button) to switch between **3D chase camera** and the top-down view. Everything else (HUD, training, weather, ghosts, map on **M**) works in both.

How it is drawn, in the spirit of 90s/2000s racers: a low-resolution picture (180p to 540p, chunky nearest-neighbour pixels), distance fog, billboard trees, depth-sorted walls. The road, grass and kerbs are the *same world map* as the top-down view, projected with true perspective, so the Nordschleife you see in 3D is the real GPS layout. Night uses lamp posts, a headlight cone and a starry sky; rain uses streaks, grey fog, spray and a shorter view distance. The camera lags a little behind the car's heading, so you see it slide in the wet.

Limits: the GPS trace has no altitude, so the world is **flat** (no crests or the Flugplatz jump). The tyre/armco walls are coloured quads, not textured. I could not open the two reference repos you linked (the fetch tool wouldn't load them), so this is built from the standard pseudo-3D techniques rather than their code. The rear-view car shipped in `assets/` is a **placeholder**: replace `car_back.png` and `car_back_lights.png` with yours.

**Assets:** see `assets/README_ASSETS.md` for the full list (rear car, lights version, left/right turning versions, trees, sky panoramas, grass and asphalt tiles). All optional.

## If the car is stuck and won't finish a lap

Raising the learning rate does **not** fix this, and usually makes PPO less stable. I measured it: three learning rates (1e-4, 3e-4, 1e-3) all stalled at the same spot, about 1.7 km in (8% of the lap), for 100+ simulated minutes. The car arrived at a 100 px-radius corner flat out (about 190 of 200 px/s) with the brake untouched, and nothing in the reward told it that braking early was worth it. What helps, in order:

1. **Study this driver** (Controls -> Learn from real drivers). With it, my headless test completed full laps from the start line (about 4:10) within roughly 90 simulated minutes, and noisy laps finished too.
2. **"Penalty: too fast for the road ahead"** (default 0.3, new). It is computed from the centre sensors only: if the car couldn't stop within the clear road it sees, it loses reward. Without it, braking has to be discovered by luck.
3. **Exploration noise**. It now adapts: it shrinks when the noise-free test drive improves and grows when stuck. Shrinking it on a timer instead was worse (the car froze at the corner), so that is not how it works.
4. **Champion brain.** Every 3 updates the AI does a noise-free test drive from the line and keeps the best brain. If learning collapses it rolls back. The HUD shows test drive %, champion % and the noise level.
5. Learning rate is now 1.5e-4 with bigger batches (1536 steps, 4 epochs). Lower is safer, higher is not faster.

Also: **Train mode drives with noise, so it fails long before the test drive does.** Judge progress by the test-drive/champion lines, or switch Mode to **Drive**. Honest limits: in my tests, learning from scratch with these settings still needed around 150+ simulated minutes to pass that corner. RL is random, and results depend on conditions (rain and night are slower).

## What you see

- **Chase view** (Zoom slider). **M** = full map: green = furthest ever reached, red dots = recent crashes, km markers.
- **Speed:** 0.5×, 1× (real time), 2× ... 32×, Max.
- **Top-left:** attempt, this run, best run, average of last 10 (metres), furthest %, laps, a chart of every attempt, and current conditions.
- **Bottom-left:** speed (km/h, to scale with the 2.4 px = 1 m map), steering, throttle/brake, slip, ghost gap. **Bottom-right:** minimap.
- Keys: **M** map, **P** or **Space** pause (Space is the handbrake while you drive), **D** drive yourself, **V** view, **Tab** controls.
- **Export attempts (CSV):** every attempt with distance, conditions and result, for your own graphs.

## How it learns

- **Observation (14 numbers):** 9 ray distances, speed, steering angle, slip angle, wetness, night.
- **Actions:** steering and throttle/brake (Gaussian, learned).
- **Reward:** progress along the road + a little speed - off-centre penalty - "too fast for the road ahead" penalty; big penalty for hitting a barrier. Stalling or going backwards ends the attempt.
- **Smart restart (on by default):** after a crash the car often restarts a bit before the furthest point it reached, so a 20 km lap is learnable. It still only sees its sensors. Untick for purer (much slower) learning.
- A "lap" counts only if started from the line.

## Do I need an API key?

**No.** The optional **LLM coach** supports free tiers only: **Gemini** (`gemini-flash-latest`, key from aistudio.google.com/apikey) and **Groq** (`openai/gpt-oss-120b` or `-20b`, key from console.groq.com/keys). Put one key in `.env` (made from `.env.example` on first run) and restart. Without a key a simple rule-based coach answers. I could not call those APIs from here, so if a model name is rejected, change it in `.env`.

## Files

```
start.bat, server.py, .env.example      launcher, local server (stdlib only), optional keys
assets/car.png, car_lights.png          your sprites
web/js/env.js         time of day + weather + curriculum stages
web/js/sim.js         physics (grip, slip), sensors, reward
web/js/rl.js          neural nets + PPO from scratch
web/js/trainer.js     rollouts, smart restart, per-condition stats, ghost recording
web/js/human.js       keyboard driving, lap recorder, learning from your laps
web/js/drivers.js     reference lap times, teacher driver, ghost clock, imitation learning
web/js/render.js      tiles, night lighting, rain, spray, sprite swap, optional art loader
web/js/view3d.js      3D chase camera (perspective ground, sprites, fog, night, rain)
web/js/main.js        game loop, HUD, controls
web/js/track.js, nordschleife.js   track from GPS
tools/headless_test.js   optional: node tools/headless_test.js 1 nord imitate night rain
```

## Tweaks

| Want | Change |
|---|---|
| Braking pressure | `wSafe` slider, or `vSafe` formula in `Car.step` (`sim.js`) |
| Rain strength | `0.42` grip loss in `sim.js` (`grip = 1 - 0.42 * ENV.wet`) |
| Night sensor loss | `0.42 * ENV.night` in `Car.sense` |
| Day length | `DAY_SIM_SECONDS` in `env.js` |
| Curriculum rule | `checkCurriculum` in `trainer.js` |
| Faster/slower car | `MAXV`, `AY`, acceleration in `sim.js` |
| Darker night / more lamps | `buildLamps` and `Lighting.draw` in `render.js` |

## Credits

Nordschleife centreline: github.com/maciejb2k/nurburgring-nordschleife-geojson (no licence file in that repo; check before publishing publicly). Lap times: see `web/js/drivers.js`. Everything else is generated in code. Car images: yours.

## Troubleshooting

- Old saved brains from the previous version won't load (the AI now has more inputs); start a new one.
- Lights car looks the wrong size: make `car_lights.png` the same canvas size as `car.png`.
- Choppy on a weak PC: lower Zoom or use a lower speed. Night uses a half-resolution light layer.
- Ctrl+F5 after editing files in `web/` or `assets/`.
