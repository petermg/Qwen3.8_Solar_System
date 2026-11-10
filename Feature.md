# Solar System — Photorealistic 3D Interactive

A single, self-contained HTML file (`solar-system.html`) that renders an interactive, photorealistic
3D solar system with **astronomically accurate** orbital positions, Earth rotation, and a real star
catalog. No build step, no dependencies beyond the Three.js CDN — open the file and it runs.

---

## 1. Rendering & Platform

| Aspect | Detail |
|---|---|
| Engine | **Three.js r160** loaded via CDN `importmap` (`three` + `three/addons/`) |
| Renderer | `WebGLRenderer` — antialias, `preserveDrawingBuffer`, capped pixel ratio (≤ 2) |
| Color pipeline | ACES Filmic tone mapping (exposure 1.1), sRGB output |
| Lighting | `PointLight` at the Sun (origin) + faint space `AmbientLight` |
| Camera | `PerspectiveCamera` (55° FOV, near 0.1 / far 6000) with `OrbitControls` (damping) |
| Fallback | A "WebGL unavailable" screen is shown if the context cannot be created |
| File | One standalone `.html` — styles, markup, and all JS are inline |

**Texture sources** (all verified HTTP 200, CORS `*`): NASA/JPL-style maps from the
`greengem/threejs-solar-system` and `mrdoob/three.js` GitHub repos — Sun, all 8 planets, Moon,
Saturn's rings, Earth day/night/specular/normal/clouds, and an 8192×4096 DSS star field.

**Resilience:** every texture has a **procedural canvas fallback** (banded gradients + noise; a
dedicated concentric-ring generator for Saturn). If a texture 404s, the scene still renders and a
warning notes which fell back. A **loading screen** shows a progress bar and `n/17` texture count.

---

## 2. The Bodies

| Body | Notes |
|---|---|
| **Sun** | Animated plasma shader (granulation + flare hotspots), soft hazy corona sprite, 3 CME clouds |
| **Mercury → Neptune** | Real surface textures, axial tilt, individual rotation period |
| **Earth** | The showcase — custom day/night shader (see §4), separate cloud layer, aurora borealis |
| **The Moon** | Orbits Earth (27.32-day period), tidally-locked spin, real texture |

Each body carries: name, radius, orbital distance, axial tilt, rotation period, a fact table
(diameter, day length, year, moons, gravity, mean temp), and a descriptive paragraph.

---

## 3. True Keplerian Orbital Mechanics

Positions are **not** circular approximations — they are computed from real orbital elements.

- **JPL J2000 Keplerian elements** for all 8 planets: semi-major axis `a`, eccentricity `e`,
  inclination `i`, longitude of ascending node `Ω`, argument of periapsis `ω`, mean longitude `L`,
  plus the rate of change of each per Julian century (`aD, eD, iD, OD, wD, LD`).
- **Newton–Raphson solver** for Kepler's equation `M = E − e·sin(E)` (converged to 1e-14).
- Full orbital-plane → ecliptic rotation chain (`ω → i → Ω`), mapped to the scene frame
  (scene `y` = ecliptic north, so the system lies in a plane).
- **Elliptical orbit rings** drawn from the same elements (256 points), regenerated when scale changes.

> Verified: Earth's heliocentric longitude is ≈ 100.4° at J2000, ≈ 43.7° today, ≈ 98.7° in 2100 —
> matching ephemeris reference values.

---

## 4. Accurate Earth Rotation & Day/Night

Earth's spin is **accurate to the date and time**, so the continents *and* the day/night terminator
sit in their true orientation for any instant.

- `earthRotationY(T, P)` derives the phase from:
  - **Greenwich Mean Sidereal Time (GMST)** — the IAU polynomial,
  - **the Sun's right ascension** — from Earth's current orbital position,
  - **the obliquity of the ecliptic** (23.44°).
- It locates the **sub-solar point** (where the Sun is overhead, at `SunRA − GMST`) and solves for
  the exact spin that makes that point face the Sun — automatically handling the axial tilt and the
  seasonal declination (terminator is vertical at equinoxes, angled at solstices).
- Texture mapping was measured from the geometry: Greenwich (0°) = body `+X`, east longitude φ =
  `(cos φ, 0, −sin φ)`.

**Earth shader** additionally renders: a smooth **day→night blend**, **city lights** on the dark
side (from the NASA night-lights map), an **ocean specular sun-glint**, a blue **atmospheric rim**,
and an animated **Aurora Borealis** (green/teal/purple curtains confined to the polar oval,
night-side only).

> Verified: the Greenwich meridian reads **night** at 00 UTC, **dawn** at 06 UTC, **noon** at 12
> UTC, **dusk** at 18 UTC (dot values ±0.96 at midnight/noon, ±0.28 at dawn/dusk). The sub-solar
> point tracks westward at 15°/hr.

---

## 5. Accurate Star Field

Two layers, both in the same ecliptic frame as the planets:

1. **Real DSS photographic sky** — the 8192×4096 equirectangular map on a huge inverted sphere,
   **dimmed** (`0x555555`) so it reads as the Milky Way band + faint haze.
2. **Real star catalog** (`STAR_CATALOG`) — the brightest ~71 naked-eye stars, each with J2000
   **right ascension, declination, V-magnitude, and B−V color index**.
   - **Position:** RA/Dec → ecliptic lon/lat → scene coordinates (obliquity 23.44°), at radius 2500,
     so every constellation sits in its **true position relative to the ecliptic plane**.
   - **Size:** derived from apparent magnitude (Sirius largest, faint stars smaller), clamped 0.75–6.5 px.
   - **Color:** derived from B−V via a piecewise-linear ramp (blue-white → white → yellow → orange → red).
   - **Render:** a `THREE.Points` cloud with a custom shader (gaussian core + halo, additive blending).

> Verified: Sirius, Arcturus, Antares, Regulus, Spica, etc. all land at their true ecliptic
> coordinates (e.g. Regulus at ecliptic latitude ≈ 0.5° — it famously lies on the ecliptic).

---

## 6. The Sun (Animated)

- **Living plasma surface** — shader-driven granulation and moving flare hotspots over the real map.
- **Soft corona** — a procedurally-built radial-gradient sprite (no hard edges), with subtle
  streamer blobs and limb active regions; it **pulses and rotates** over time.
- **Coronal Mass Ejections** — 3 organic billowing plasma clouds that erupt from the limb in
  random directions, expand, brighten, and fade on a loop.
- **Rotation** — the Sun spins on a ~25.4-day solar period.

---

## 7. Time Control

- **1× = real time** — the simulation clock is **anchored to the wall clock**:
  `simMs = clockBase.sim + (Date.now() − clockBase.wall) × timeScale`.
- **Independent clock driver** — a `setInterval` (60 ms) recomputes `simMs` and re-renders, so the
  sim advances in real time **even when the tab is backgrounded** (rAF throttling-proof). The rAF
  loop adds smooth rendering and camera/label updates when visible.
- **Speed range 1× → 1 billion ×** — a **logarithmic** slider (0–100 → `10^(s·9/100)`) plus an
  exact numeric input.
- **Live speed hint** — `speedHint(v)` shows what the speed means in real terms: Earth's orbit time
  when it's > 1 s, otherwise Neptune's (the slowest planet), e.g. "Earth orbit: 3.6 d" or "Neptune
  orbit: 5.2 s".
- **Pause / Play** — freezes the clock (re-anchors so there's no jump on resume).

---

## 8. Date & Time Selection

- **Date + time pickers** — set the simulation to any instant; changing them jumps the sim and
  pauses so it stays put.
- **"📅 Set to Today / Now"** — a prominent green button that snaps the sim to the exact current
  system date/time.
- **Ticking clock display** — the pickers refresh ~4×/sec while running (suppressed while the user
  is editing them).
- **`setSimDate(ms)`** — re-anchors the clock and immediately repositions + renders (works even if
  the render loop is throttled).

---

## 9. Camera & Interaction

- **Orbit / zoom / pan** via `OrbitControls` (drag, scroll, right-drag), with damping.
- **Click to focus** — click any body (or use the bar/keys) and the camera **flies in** and then
  **travels with** the body, keeping your orbit/zoom/pan offset as the planet moves underneath.
- **Hover tooltips** — a floating label tracks the cursor over any pickable body.
- **Fly-in animation** — eased camera transition that re-targets the *current* position of a moving
  body each frame (converges on the planet, not its click-time location).
- **Reset view** — animated return to the home vantage.
- **Planet quick-jump bar** — a bottom bar with a button per body (Sun through Neptune).

---

## 10. Display Toggles

| Control | Effect |
|---|---|
| **Planet size** | 0.5×–2.5× scale slider (mesh, rings, clouds, Moon) |
| **Realistic scale** | Switches to true-to-scale radii and **AU-true distances** (regenerates orbit rings) |
| **Orbit rings** | Show/hide the elliptical orbit lines |
| **Labels** | Show/hide projected HTML name labels for every body |
| **True colors** | Full ACES exposure + white surfaces vs. a dimmed, blue-gray desaturated mode |

---

## 11. Educational Features

- **Info panel** — clicking a body shows its fact table (diameter, day, year, moons, gravity, temp)
  plus a descriptive paragraph.
- **Quizzes** — each planet has a multiple-choice question in its info panel; answers are
  color-coded (green = correct, red = your pick) on click.
- **Guided tour** — a "▶ Guided tour" button auto-visits every body in order (Sun → Neptune),
  focusing and holding 9 s each; click again to stop.
- **HUD** — a help panel explains the controls and lists keyboard shortcuts.

---

## 12. Keyboard Shortcuts

| Key | Action |
|---|---|
| `Space` | Pause / Play |
| `R` | Reset view |
| `1`–`9` | Jump to body (1 = Sun … 9 = Neptune) |
| `L` | Toggle labels |
| `O` | Toggle orbit rings |

(Shortcuts are ignored while typing in an input field.)

---

## 13. Architecture Notes

- **Single file** — everything (CSS, HTML, JS) is inline; the only network dependency is the
  Three.js CDN and the texture images.
- **Two render drivers** — rAF for smooth visible rendering + a 60 ms `setInterval` that keeps the
  sim, camera-follow, labels, and clock in sync when rAF is throttled.
- **Shared update functions** — `updatePositions(T, now)` (all bodies, Earth spin, Moon, Sun) and
  `updateCosmos(T, now, dt)` (corona + CMEs) are called from both drivers to avoid drift.
- **Procedural fallbacks** — the scene degrades gracefully offline (canvas-generated textures).
- **Debug hooks** — `window.__ss` (renderer, scene, camera, `setSimDate`, `keplerPosition`, etc.)
  and `window.__ssStars` (the star `Points`) are exposed for inspection; harmless in production.
- **Syntax-clean** — validated with `node --check`; zero console errors at runtime.

---

*File: `solar-system.html` · Three.js r160 · No build step · Open and run.*
