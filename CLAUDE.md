# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**颈部律动 (Neck Rhythm)** — browser-based head-movement rhythm game. Single HTML file (~1300 lines), zero build step, double-click to play. Arrows converge from screen edges toward center; player moves head (webcam) or presses keys to interact. Designed for office workers to exercise their neck.

Two game mechanics (selectable on start screen):
- **都来都来 (Stretch)**: arrow enters zone ring → player moves head in **same** direction → "eat" → score. Any time while arrow overlaps the ring.
- **别来沾边 (Rebel)**: arrow reaches zone boundary → player must already be in **opposite** direction → "block" → score. **No grace period** — if head isn't opposite when arrow touches the ring, instant Miss.

## How to Run

```
Open index.html in Chrome/Edge (recommended for file:// camera access)
Or: python -m http.server 8080 / npx serve .
```

Camera mode needs HTTPS or localhost/file in Chrome. Keyboard mode works anywhere.

## Architecture

Everything in `index.html`: inline CSS → inline HTML → inline `<script>` (not module).

### Class hierarchy

1. **`AudioEngine`** — Procedural music via Web Audio API (kick/snare/hi-hat/bass oscillators). BPM set per difficulty. `AudioContext` created fresh each game via `init()`, destroyed via `destroy()`. Timing via `ctx.currentTime - startTime`. `_started` flag prevents pre-start time leakage.

2. **`Particle` / `ParticleSystem`** — Simple particle pool. `emit(x, y, color, count, speed, life)` for hit/miss effects.

3. **`BeatMapGenerator`** — Procedural note patterns. Config: `{bpm, noteDensity, travelBeats, zoneRadius, diagonalChance, songDuration}`. Enforces `minGap ≥ 0.4` beats between notes to prevent overlapping zone entries. Rapid-fire sequences only on hard mode. Returns sorted `[{beat, direction, isDiagonal}]`.

4. **`Game`** — Core engine. Owns Canvas rendering. Key state:
   - `notes[]` — full beat map. `activeNotes[]` — currently on screen.
   - `state`: idle → countdown(3s) → playing → finished
   - `mechanic`: `'stretch'` (都来都来) or `'rebel'` (别来沾边)
   - **都来都来 hit detection**: each frame, filter `activeNotes` where `inZone && direction.id === headDirection` → hit closest to center. Score by distance ratio.
   - **别来沾边 hit detection**: on zone entry (`!wasInZone && inZone`), check if `headDirection` is OPPOSITE of arrow direction **immediately**. If yes → hit. If no → instant miss. No grace period. `OPPOSITE = {0:2, 1:3, 2:0, 3:1, 4:6, 5:7, 6:4, 7:5}`.
   - `playStartTime` prevents song-end before 5s of actual play.
   - `draw()`: radial grid → warning zone (rebel only, pink dashed ring at `zoneRadius*1.4`) → hit zone ring → direction labels (highlight active) → notes → particles.

5. **`HeadTracker`** — MediaPipe Face Mesh (CDN global `window.FaceMesh`). Key parameters:
   - Reference: eye inner corners (lm 33, 362) midpoint — more stable than face oval
   - Nose offset from eye center, **normalized by inter-eye distance** (scale-invariant across different face distances)
   - Direction: `Math.atan2(dy, dx)` — user perspective (turn head right = direction right)
   - **Threshold: 0.020** — requires deliberate head movement (was 0.012, too sensitive)
   - Direction hold: 200ms persistence after last detection
   - Face loss timeout: 350ms before clearing direction
   - Adaptive neutral: drifts at rate 0.01 when head is at rest
   - Camera resolution: **320×240** for faster processing
   - Calibration: 1000ms or 20 samples, **3s hard timeout** via `setTimeout`
   - `_computeOffset()` wrapped in try-catch — returns null on bad landmarks
   - `stop()` clears calibration timer, face mesh, camera tracks, and resolves pending calibration promise

6. **`KeyboardInput`** — WASD and Arrow keys → `getDirection()` returns direction ID (0-7). Diagonals via two-key combos.

7. **`UIManager`** — Screen transitions. `showGame()` hides loading + calibration + results, shows HUD. `showBackBtn()`/`hideBackBtn()` for the in-game return button.

8. **`App`** — Orchestrator. Key flow:
   - `_startGame()`: cleanup → `audio.init()` → if camera: show calibration screen FIRST, then `_withTimeout(init, 6s)` + `_withTimeout(calibrate, 5s)` → `showGame()` + `showBackBtn()` → `game.start()` → `_gameLoop()`
   - `_camAborted` flag: set by skip button, checked after init/calibrate — aborts entire camera flow
   - Safety: `_starting` guard with 20s auto-reset timer; `_goToMenu()` resets `_starting`
   - Camera failure → auto-switch to keyboard mode with alert

### Data flow (per frame):
```
KeyboardInput.getDirection() ─┐
HeadTracker.currentDirection ─┤ → App._gameLoop() → Game.update(dt, headDir) → Game.draw()
```

### Direction mapping (`DIRS`):
| id | name | angle | color | label |
|----|------|-------|-------|-------|
| 0 | up | -π/2 | #00e5ff | ↑ |
| 1 | right | 0 | #ff2d95 | → |
| 2 | down | π/2 | #ffd740 | ↓ |
| 3 | left | π | #b388ff | ← |
| 4-7 | diagonals | ±π/4, ±3π/4 | various | ↗↘↙↖ |

## Key Design Decisions

- **Zone-based, not timing-based**: notes are hit when arrow overlaps the ring AND head direction matches. No precise rhythm timing. Health-first design.
- **别来沾边 is instant**: check at zone boundary only, no grace period. Player must anticipate.
- **Camera resolution 320×240**: faster MediaPipe processing, lower latency.
- **Calibration hard timeout 3s**: prevents infinite hang if face detection fails.
- **Camera init 6s timeout**: `Promise.race` wrapper. Skip button sets `_camAborted` flag + calls `headTracker.stop()`.
- **`_starting` guard + 20s safety reset**: prevents double-start; auto-recovers from stuck async flows.
- **Single file, CDN-only**: MediaPipe Face Mesh from jsDelivr. No build tools.
- **Adaptive neutral calibration**: continuously drifts resting position, no recalibration needed.
