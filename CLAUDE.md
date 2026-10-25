# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Speed Blitz is a 3D browser game. You are a superhero in Blitz City, a town full of people and cars. You can run at 200 mph, freeze time, speed blitz everyone nearby, and use super strength (punch, ground slam, lift and throw cars). The whole game is a single self-contained HTML file.

## Running the Game

```
open speed-blitz.html
```

No build step. Three.js 0.160 (plus `BufferGeometryUtils`) loads from jsDelivr through an import map, so an internet connection is needed.

## Architecture

**Two canvases**: `#gl` holds the Three.js view. `#hud` sits on top and handles all input (pointer lock on desktop, touch joystick/buttons on phones) plus the 2D HUD, comic popups and menus.

**State**: `state` is `'title'`, `'play'`, `'paused'` (pointer lock lost) or `'busted'` (health hit 0; click after 1.5 s runs `resetGame()`). `frame()` runs `stepPlay(dt)` during play. When `frozen` is true the world step `wdt` is 0, so people, cars, police, particles and flying bodies stop, while the hero, rings, bolts and popups still use real `dt`. Anything launched during a freeze hangs in the air until time restarts. `freezeE` drains while frozen and recharges otherwise. The frozen look comes from blending the lights, fog and sky toward the `ICE_*` colors (`freezeMix`), not from a CSS filter.

**Performance**: the game is limited by pixels, not scene size (about 63 draw calls and 90k triangles a frame). `QUALITY` presets (`low`/`medium`/`high`, saved as localStorage `speedBlitzQuality`, changed with the button on the title screen, which reloads the page because antialias is fixed when the renderer is created; `?q=low` overrides) set render resolution (`dprCap`), HUD resolution (`hudDPR`), shadows, shadow map size, tree shadows and antialias. The renderer asks for `powerPreference: 'high-performance'` so dual-GPU Macs use the discrete GPU. The city is merged into `NCH`×`NCH` chunk meshes so off-screen chunks get culled, each car is one merged mesh (`carGeometry()`), and HUD vignettes are painted once per resize (`vignette()`). `frame()` lowers `dprCap` further if the frame rate stays under ~42 fps, and splits long frames into two `stepPlay()` steps so low frame rates don't turn into slow motion. P (or `?fps`) shows an FPS meter with game/draw/HUD milliseconds and draw calls.

**Police**: KOs and smashed cars call `addHeat()`. `heat` decays after 10 s of calm, and `STAR_HEAT` turns it into 0–5 wanted `stars`. `updatePolice()` dispatches police cars (`police`, also inside `cars`) up to `COPS_FOR_STARS[stars]`, spawning them 110–170 m away. `updatePoliceCar()` steers straight at the hero, slides along buildings, rams for damage, and at 18 m parks (`'parked'`) and deploys two officers with `deployCop()`. Officers are the last `NCOP` entries of `people` (`p.cop`, with hat and badge parts); their `'chase'` state runs at the hero and fires `copShoot()` (pooled `tracer()` lines, hit chance drops with hero speed and distance) into `damageHero()`. With 0 stars, cars drive off and officers walk away and vanish. Unused cops and police cars sit in `'gone'` with `t = Infinity`. The hero has `hp` that regenerates 3 s after the last hit; a siren tone gets louder as police cars get closer.

**Town**: `N`×`N` blocks of `CELL` meters; roads run along `gridLine(i)`. All buildings are merged into one mesh (`addBox()` → `mergeGeometries`) with a tiling window texture. `blockB[i*N+j]` keeps each block's building boxes for `findBuilding()`, `collide()` (push out of walls) and `groundAt()` (rooftops count as ground). Parks have trees (instanced).

**People**: `NP` townspeople drawn with one `InstancedMesh` per body part (`PARTS`), matrices built in `syncPeople()`. Each has a center `c` and quaternion `q`. States: `walk` → `panic` (runs away, arms up) → `fly` (`stepBody()` physics, can knock others over via `knockOthers()`) → `ko` (lies flat) → `gone` (poof, then `spawnPerson()` far from the hero). `launch()` sends any person or car flying and counts the KO.

**Cars**: `NC` regular cars drive the road lanes (`updateDrivingCar()`, turning at intersections); states `drive`, `held`, `fly`, `wreck`, `gone`. Each car has its own material so it can darken with damage. Every `launch()` of a car calls `hurtCar()`; a thrown car (`thrown`) also takes a hit when it crashes hard. Hit 1 makes it smoke, hit 2 sets it on fire, and hit `CAR_HITS` (3) calls `explode()`: a `fireball()`, particles, a screen flash (`blastFlash`), heat, damage to the hero within 7 m, and a 14 m blast that launches everything nearby. Cars caught in a blast get a 0.25 s `fuse`, so chain reactions go off one after another. Exploded cars stay as burning wrecks until they poof; `repairCar()` resets them on respawn. The aim bracket shows a car's damage pips.

**Hero powers**: `punch()` (left click, hold to repeat; `aimTarget()` picks what is under the crosshair and the hero lunges to it), `startBlitz()`/`updateBlitz()` (right click or Q, chains through up to 16 nearby targets with lightning `bolt()`s), `toggleFreeze()` (F), `slam()`/`shockwave()` (E), `grab()`/`throwHeld()` (G), `jump()` (Space, double jump, Shift+Space super jump). Shift ramps `hero.maxSp` up to `MAX_SPEED` (Mach 3, about 2,300 mph; `MACH` is 767 mph). Acceleration grows with speed, so 200 mph takes about 1.5 s and Mach 3 about 3 s. `updateHero()` moves in ~2 m substeps so Mach speeds can't skip through buildings or people; crashing into a wall above Mach 0.5 makes a shockwave and cuts speed. Running into people above ~50 mph launches them, with a wider wake at higher speeds, and super-speed hits are capped at `MAX_KNOCK` so things don't fly off at Mach speeds. Crossing Mach 1/2/3 gives a boom, and `updateVapor()` shows a vapor cone near the sound barrier. Animation, speed lines and most of the FOV change scale to `SPRINT_SPEED` (200 mph); the camera pulls back further at Mach speeds. The hero can roam `HALF + 2000` out into the grass.

**Testing**: `?test` starts in play with a scripted hero that uses every power, runs 150 sim steps per rendered frame, keeps health at 40 or more, and writes `T<time> KO<n> CARS<n> stars<n> cops<n> police<n> hp<n>` into `document.title`. `?test&still` instead sets a high wanted level and then stands still, so you can watch police arrive and shoot. `?test&mach` runs straight from far outside town at full speed into the city and reports top speed and whether the hero ever ended up inside a building. `?test&boom` hits one car three times next to an already-damaged car and reports when each exploded. `?test&bench` runs the scripted hero at one step per frame and writes FPS and timings into the title instead. Uncaught errors go to `document.title` as `ERR ...`, so headless `--dump-dom --virtual-time-budget=30000` runs can check both.
