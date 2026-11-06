# Void Sentinel

A fast, polished 2D space shooter that runs in the browser. It is a single HTML file with no build step and no dependencies. Everything you see (the fighter jets, asteroids, Earth, explosions and sound effects) is generated in code when the page loads.

## Play it

1. Download `void-sentinel.html` (rename it to `index.html` if you want to host it).
2. Open it in any modern browser.

To host it for free on GitHub Pages: push the file to a repository, then go to **Settings → Pages**, choose your branch and the root folder, and save. Your game will be live at `https://<your-username>.github.io/<repo-name>/`.

## Controls

| Action | Keyboard | Mouse / Touch |
| --- | --- | --- |
| Move | `W A S D` or arrow keys | Move the mouse, or drag with a finger |
| Fire | Automatic | Automatic |
| Rocket | `Q` | Tap the rocket button (bottom left) |
| Shield | `E` | Tap the shield button (bottom right) |
| Pause | `P` or `Esc` | Tap the pause button (top center) |
| Mute / unmute | `M` | - |
| Start / restart | `Enter` or `Space` | Tap the Launch button |

## How the game works

### Goal
Survive as long as you can and score as high as possible. You have 3 lives. Difficulty rises every 22 seconds (a new wave).

### Enemies

| Enemy | Health | Behavior |
| --- | --- | --- |
| Raider (magenta jet) | 1 | Weaves side to side in a sine wave while descending |
| Striker (amber jet) | 2 | Dives fast and steers toward your position |
| Gunship (violet jet) | 7 | Hovers near the top and fires a 3-shot aimed spread |
| Asteroid (large / medium / small) | 10 / 4 / 1 | Tumbles and bounces off the side walls. Large splits into 3 medium, medium splits into 2 small |

Every 24 to 34 seconds an **asteroid shower** sends 9 rocks in quick succession.

### Power-ups (dropped by destroyed enemies)

| Pickup | Effect |
| --- | --- |
| **P** | Weapon level up (single shot, double shot, triple spread; max level 3) |
| **S** | One-hit shield |
| **R** | +1 rocket (max 5) |

### Abilities
- **Rocket** (3 per run): launches a missile that accelerates upward and explodes on contact or near the top of the screen. The blast deals 7 damage to everything within 190 pixels.
- **Shield** (3 per run): absorbs the next hit.

### Scoring
Each enemy has a base score (100 for a raider, 180 for a striker, 500 for a gunship, 60/150/300 for asteroids). Chaining kills without a 2-second gap raises your **combo multiplier** by one every 4 kills, up to ×8. Your best score is saved in the browser (`localStorage`, key `voidSentinelBest`).

### Taking damage
Getting hit costs a life, drops your weapon one level and gives you 2.2 seconds of invulnerability (the jet blinks). On your last life the screen edges pulse red.

## Code walkthrough

All code lives inside one immediately-invoked function in the `<script>` tag. Here is how it is organized, in file order.

### 1. Setup and glow sprites
```js
const cv = document.getElementById('c'), ctx = cv.getContext('2d');
```
The game draws to a full-screen `<canvas>` using the 2D API. `glow(color)` builds a small radial-gradient sprite once and caches it. Almost every light effect (bullets, sparks, engine flames, power-ups) is that sprite drawn with `globalCompositeOperation = 'lighter'`, which adds light instead of covering what is behind it. This is cheaper than `shadowBlur` and looks softer.

### 2. Procedural asset factory
Instead of image files, the art is drawn once at load time onto off-screen canvases (`mk(w, h, fn)`), then reused every frame with `drawImage`:

- `makeJetP()` draws the player's fighter: a symmetric body path with metal gradients, a neon outline, wing stripes, intakes, missiles, a glass canopy and engine nozzles. A `sym()` helper draws each half-shape and mirrors it.
- `makeJet(kind, color)` draws the three enemy jets the same way.
- `makeAsteroid(radius, seed)` builds an irregular polygon from jittered points, then fills it with a gradient, clips and adds craters, adds side shading, and finishes with mineral glints. It uses `mulberry(seed)`, a tiny seeded random generator, so each variant is repeatable. 4 variants per size are created.
- Sprites are rendered at 2× (`SC = 2`) so they stay sharp on high-DPI screens.

### 3. Backdrop and the Earth
`resize()` handles the window size, device pixel ratio (capped at 2) and pre-renders the background to an off-screen canvas, so each frame only needs one `drawImage`.

The Earth (`earth2`) is computed per pixel on a half-resolution canvas:
1. Each pixel inside the planet circle is mapped to a point on a 3D sphere.
2. A fractal value-noise function (`fb`) samples that point to get terrain height. Low values become ocean (dark to shallow blue), high values become land (green to rock, with desert areas from a second noise layer).
3. A third noise layer produces clouds that are lit by the sun.
4. Lighting uses the dot product of the surface normal and a sun direction. The dark side shows faint city lights, the ocean gets a specular sun glint, and a Fresnel term adds the blue atmosphere at the edge.
5. The result is cached per screen size, so it is only computed once.

### 4. Input and audio
- Keyboard state is stored in a `keys` object; pointer position is stored in `ptr`. Keyboard movement uses acceleration with damping; pointer movement eases the jet toward the finger (offset upward on touch so your finger does not hide the ship).
- Sound is synthesized with the Web Audio API (`sfx()`): oscillators for shots and pickups, and a filtered noise burst plus a falling sawtooth tone for explosions. No audio files are used.

### 5. Game state
Everything is plain variables and arrays: `bullets`, `ebul` (enemy bullets), `enemies`, `parts` (particles), `rings`, `pups`, `mis` (missiles) and `streaks`. A `state` string (`'menu'`, `'play'`, `'over'`) decides what `update()` does.

### 6. The update step
`update(dt)` runs once per frame with the elapsed time in seconds, so speed is the same on 60 Hz and 144 Hz screens (`dt` is capped at 1/30 s to avoid jumps after a tab switch). In order, it:
1. Scrolls the stars and ages particles, rings and shooting stars.
2. Moves the player, auto-fires every 0.15 s and picks the bullet pattern from the weapon level.
3. Spawns enemies. The spawn gap shrinks as `diff = 1 + time / 22` grows, and the wave banner appears when the wave number changes.
4. Moves bullets, missiles and enemies (each type has its own movement branch).
5. Checks collisions with simple circle-distance tests: bullets vs enemies, enemy bullets vs player, enemies vs player, pickups vs player.
6. Handles the combo timer.

### 7. Particles and explosions
`explode(x, y, color, big)` layers several effects to build one blast:
- a white flash and an orange fireball (large additive glow sprites),
- slow fire puffs and fast colored sparks,
- streak particles drawn as short lines along their velocity,
- grey smoke puffs that expand and fade (drawn with normal blending, so they darken instead of brighten),
- tumbling debris shards with gravity,
- two expanding shockwave rings, a brief full-screen flash and camera shake.

Each particle has a `kind` (`streak`, `smoke`, `deb`, or the default glow) that the renderer uses to pick how to draw it.

### 8. Rendering
`render()` draws back to front: backdrop, parallax stars (3 layers that also shift slightly with your position), pickups, enemies, enemy bullets, player bullets and missiles (additive), the jet with a soft shadow and engine glow, then particles and rings, a flash overlay, the low-health red pulse and a vignette. The backdrop is drawn at 62% opacity each frame so moving objects leave a faint motion trail. Screen shake is a random translate whose size decays every frame (disabled if the user prefers reduced motion).

### 9. HUD and menus
The HUD is regular HTML and CSS on top of the canvas (hexagon buttons and slanted panels built with `clip-path`), so the text stays sharp. `hud()` only writes to the DOM when a value actually changes. The start, pause and game-over screens are overlay `<div>`s toggled with the `hidden` attribute.

### 10. Main loop
```js
function frame(now){
  const dt = Math.min(.033, (now - last) / 1000); last = now;
  if (!paused) update(dt);
  render();
  requestAnimationFrame(frame);
}
```

## Project structure

```
void-sentinel.html   # the entire game: HTML + CSS + JavaScript
README.md            # this file
```

## Customizing

| Want to change | Look for |
| --- | --- |
| Difficulty ramp | `diff = 1 + time / 22` and the `st = Math.max(.32, 1.05 - diff * .09)` spawn timer |
| Fire rate | `p.cool = .15` |
| Starting lives / rockets / shields | `newPlayer()` and the `rk`, `sh` variables |
| Colors | the `C` palette object and the CSS variables at the top of `<style>` |
| Enemy stats | the `spawn()` function and `AHP` / `ASC` tables for asteroids |

## Notes

- Written in plain JavaScript. It can be ported to TypeScript by adding types to the entity objects (player, enemy, particle) and splitting the sections above into modules.
- Works on desktop and mobile browsers. It uses the Google Fonts "Chakra Petch" font when online and falls back to system fonts offline.

## License

Add the license of your choice (for example MIT) before publishing.
