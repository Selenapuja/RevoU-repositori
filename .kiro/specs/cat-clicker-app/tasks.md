# Implementation Plan: Cat Clicker App

## Overview

Build a single `index.html` file that delivers the complete Cat Clicker App — inline SVG cat, CSS keyframe animations, Tailwind CDN, Vanilla ES2022 JavaScript, Web Audio API sound, click counter, idle detection, milestone messages, and full WCAG 2.1 AA accessibility. A companion `test.html` runs all six property-based tests with fast-check via CDN.

---

## Tasks

- [x] 1. Scaffold `index.html` with base layout and static structure
  - Create `index.html` with `<!DOCTYPE html>`, `<head>` (charset, viewport, title, Tailwind CDN `<script>` tag), and a `<body>` using Tailwind utility classes for a centred full-height flex layout on `#fdf6ee` background
  - Add the visually-hidden `#aria-live` region (`aria-live="polite"`, `aria-atomic="true"`, `class="sr-only"`)
  - Add the `#mute-toggle` button (`aria-label="Toggle sound"`, `aria-pressed="true"`, positioned `absolute top-4 right-4`)
  - Add the `#message-text` paragraph with the default idle text "Click the cat to pet it!" and `role="status"`
  - Add the `#cat-wrapper` div (`tabindex="0"`, `role="img"`, `aria-label="Pet the cat"`, `cursor-pointer`, focus ring classes)
  - Add the `#counter-display` paragraph with `<span id="counter-value">0</span> pets`
  - Add the `#milestone-msg` div (`aria-live="assertive"`, hidden by default)
  - Add an empty `<style>` block (for keyframes and fallback rules) and an empty `<script>` block
  - _Requirements: 1.1, 1.2, 1.5, 2.3, 6.1, 6.3, 9.1, 9.3_

- [x] 2. Inline SVG cat illustration and `<style>` block
  - [x] 2.1 Draw and embed the inline SVG cat inside `#cat-wrapper`
    - Produce a minimalist rounded SVG cat (`width="300" height="300" viewBox="0 0 300 300"`) with `id="cat-svg"` using ≤ 5 colour fills and ≤ 2 stroke colours
    - Include an `<img>`-style fallback `<div>` (`id="cat-fallback"`, `200×200 px`, `aria-label="Cat illustration"`, hidden by default — shown via JS if SVG unavailable)
    - _Requirements: 2.1, 2.2, 2.4, 2.6_

  - [x] 2.2 Write CSS keyframe animations and Tailwind fallback rules in `<style>`
    - Define `@keyframes` for all 8 reaction animations: `anim-purr`, `anim-surprised`, `anim-yawn`, `anim-heart-eyes`, `anim-boop`, `anim-flop`, `anim-chirp`, `anim-knead`
    - Define `@keyframes` for 2 idle animations: `anim-tail-wag`, `anim-slow-blink`
    - Add `.sr-only` utility (position absolute, clip, overflow hidden) for the aria-live region
    - Add Tailwind-CDN-unavailable fallback rules: centering, `min-width`/`max-width` for cat, `cursor: pointer`, focus ring, responsive clamp
    - Add `@media (prefers-reduced-motion: reduce) { #cat-svg, #cat-wrapper { animation: none !important; } }`
    - _Requirements: 1.7, 2.5, 3.2, 4.3, 8.2_

- [x] 3. Implement `REACTION_POOL` constant and boot-time validation
  - [x] 3.1 Define `REACTION_POOL` array with all 8 reactions
    - Add all 8 reaction objects (`purr`, `surprised`, `yawn`, `heart-eyes`, `boop`, `flop`, `chirp`, `knead`) with correct `id`, `animationClass`, `durationMs` (400–2000), `message` (≤ 280 chars), `audioClip` key, and `category` fields
    - Cover all required categories: `happy`, `surprised`, `sleepy`, `heart-eyes`, `boop`, plus `extra` entries to reach ≥ 8
    - _Requirements: 4.1, 4.3, 4.4, 7.5_

  - [x] 3.2 Implement boot-time `REACTION_POOL` validation
    - At `DOMContentLoaded`, filter `REACTION_POOL` in-place: remove any reaction where `durationMs < 400 || durationMs > 2000`
    - For each removed reaction emit `console.warn('Reaction <id> skipped: invalid durationMs <n>')`
    - _Requirements: 4.6_

  - [ ]* 3.3 Write property test P4 — duration validation filters out-of-range reactions
    - **Property 4: Duration validation filters out-of-range reactions**
    - **Validates: Requirements 4.6**
    - Uses `fc.array(fc.record({id: fc.string(), durationMs: fc.integer({min:0, max:3000}), …}))` with mixed valid/invalid entries; assert validated pool contains only reactions with `400 ≤ durationMs ≤ 2000`

- [ ] 4. Implement `AnimationEngine`
  - [x] 4.1 Implement `pickReaction()` with no-3-consecutive rule and pool exhaustion shuffle
    - Maintain `_lastReactionIds` ring buffer (max 3); before confirming a pick, reject if all 3 match current pick (re-pick from remaining)
    - Track `_played` Set; when `_played.size === REACTION_POOL.length`, Fisher-Yates shuffle a working copy, ensure first post-shuffle item differs from last pre-shuffle pick, reset `_played`
    - Return `null` if pool is empty
    - _Requirements: 4.2, 4.5, 3.7_

  - [~] 4.2 Implement `triggerReaction(reaction)` and `clearAnimation()`
    - Remove existing animation class from `#cat-svg`, force reflow via `void catSvg.offsetWidth`, then add `reaction.animationClass`
    - Remove the class after `reaction.durationMs` ms via `setTimeout`
    - `clearAnimation()` removes any currently-applied animation class
    - _Requirements: 3.2, 3.4_

  - [x] 4.3 Implement `triggerIdle(animationClass)`
    - Apply idle animation class to `#cat-svg` using the same force-reflow pattern
    - _Requirements: 8.1, 8.2_

  - [ ]* 4.4 Write property test P2 — no reaction chosen 4+ times consecutively
    - **Property 2: No reaction is chosen four or more times consecutively**
    - **Validates: Requirements 4.2**
    - Generators: `fc.array(fc.record({id: fc.string(), …}), {minLength: 2, maxLength: 20})` for pool; `fc.integer({min: 10, max: 500})` for N picks; assert no 4+ consecutive identical id in the sequence

  - [ ]* 4.5 Write property test P3 — post-exhaustion first pick differs from last pre-exhaustion pick
    - **Property 3: Post-exhaustion first pick differs from pre-exhaustion last pick**
    - **Validates: Requirements 4.5**
    - Simulate full-pool exhaustion by picking until `_played.size === pool.length`; assert next pick id ≠ last pre-exhaustion id

- [ ] 5. Implement `SoundEngine`
  - [~] 5.1 Implement audio capability detection and clip loading
    - Detect `window.AudioContext || window.webkitAudioContext` first; fall back to `document.createElement('audio').canPlayType`; if neither, set `_capable = false` and return
    - Encode at least 3 short audio clips (meow, purr, chirp) as base64 data URIs; store in `_clipData` map
    - `loadClips()` decodes each clip into `AudioBuffer` (Web Audio path) or creates `HTMLAudioElement` (fallback); on any decode/load error set `_clips[key] = null` silently
    - _Requirements: 5.2, 5.5, 5.7_

  - [~] 5.2 Implement `play(reaction)`, `stopAll()`, and `toggleMute()`
    - `play()`: no-op if `_muted`, `!_capable`, or `_clips[reaction.audioClip] == null`; play within 50 ms of call; lazily create `AudioContext` on first call (autoplay policy)
    - `stopAll()`: stop all active sources within 100 ms; called on mute activation
    - `toggleMute()`: flip `_muted`, call `stopAll()` when muting, update `#mute-toggle` `aria-pressed` and icon (🔇/🔊)
    - Default `_muted = true` at page load
    - _Requirements: 5.1, 5.3, 5.4, 5.6, 5.8_

- [x] 6. Implement `MessageDisplay`
  - [x] 6.1 Implement `show(text)` with truncation and aria-live sync
    - Truncate: if `text.length > 80`, set display text to `text.slice(0, 80) + '…'`; otherwise use verbatim
    - Write truncated/full text to `#message-text` innerHTML and `#aria-live` textContent within 100 ms of call
    - _Requirements: 7.1, 7.4, 7.6, 9.4_

  - [x] 6.2 Implement `showIdle()`
    - Set `#message-text` to `IDLE_MSG` ("Click the cat to pet it!") and update `#aria-live` to the same; called when no reaction is active
    - _Requirements: 7.3_

  - [ ]* 6.3 Write property test P6 — MessageDisplay syncs truncated text to both elements
    - **Property 6: MessageDisplay syncs truncated text to visible element and aria-live region**
    - **Validates: Requirements 7.6, 9.4**
    - Generator: `fc.string({ minLength: 0, maxLength: 300 })`; assert both `#message-text` content and `#aria-live` content equal `msg.slice(0,80)+'…'` for `msg.length > 80`, or `msg` verbatim otherwise

- [ ] 7. Implement `ClickCounter`
  - [-] 7.1 Implement `increment(onMilestone)` and DOM update
    - `_count++` then update `#counter-value` textContent within one animation frame via `requestAnimationFrame`
    - After increment, check milestone: `_count` ∈ {10, 25, 50} or (`_count ≥ 100 && _count % 100 === 0`); if true call `onMilestone(_count)`
    - _Requirements: 3.1, 6.1, 6.2, 6.4_

  - [ ]* 7.2 Write property test P1 — counter increment is always exactly +1
    - **Property 1: Counter increment is always exactly +1**
    - **Validates: Requirements 3.1**
    - Generator: `fc.integer({ min: 0, max: 1_000_000 })` for starting count; set `_count` to start, call `increment()`, assert `_count === start + 1`

  - [ ]* 7.3 Write property test P5 — milestone detection is correct for all non-negative counts
    - **Property 5: Milestone detection is correct for all non-negative counts**
    - **Validates: Requirements 6.4**
    - Generator: `fc.integer({ min: 0, max: 10_000 })`; extract `isMilestone` as a pure function; assert returns `true` iff count ∈ {10, 25, 50} or (`count ≥ 100 && count % 100 === 0`), `false` otherwise

- [ ] 8. Implement `IdleDetector`
  - [-] 8.1 Implement `start()`, `reset()`, and idle cycle
    - `start()` sets a `setTimeout` for `IDLE_THRESHOLD_MS` (10 000 ms); on fire, call `_beginCycle()`
    - `reset()` calls `clearTimeout(_timer)` then re-calls `start()` (also calls `cancel()` to clear any active idle animation)
    - `_beginCycle()` applies the current `IDLE_ANIMATIONS[_cycleIndex]` animation via `AnimationEngine.triggerIdle()`, then `setTimeout` for that animation's `durationMs`, then advances `_cycleIndex` cyclically and recurses
    - _Requirements: 8.1, 8.2, 8.4_

  - [-] 8.2 Implement `cancel()`
    - `clearTimeout(_timer)`, set `_cycling = false`, reset `_cycleIndex = 0`, call `AnimationEngine.clearAnimation()`; cancel completes within 100 ms of call
    - _Requirements: 8.3_

- [ ] 9. Wire all modules together — main click handler and event setup
  - [~] 9.1 Implement the central `handlePet()` function and attach event listeners
    - `handlePet()` orchestrates: `IdleDetector.reset()` → `const r = AnimationEngine.pickReaction()` → if `r == null` early-return (no counter increment, Req 3.7) → `ClickCounter.increment(onMilestone)` → `AnimationEngine.triggerReaction(r)` → `SoundEngine.play(r)` → `MessageDisplay.show(r.message)`
    - Attach `click` listener on `#cat-wrapper` → `handlePet()`
    - Attach `touchstart` listener on `#cat-wrapper` with `e.preventDefault()` → `handlePet()` (Req 3.6)
    - Attach `keydown` listener on `#cat-wrapper` for `Space` / `Enter` → `handlePet()` (Req 9.2)
    - Attach `click` listener on `#mute-toggle` → `SoundEngine.toggleMute()`
    - _Requirements: 3.1, 3.3, 3.6, 9.2_

  - [~] 9.2 Implement `onMilestone(count)` callback and milestone message display
    - Show the milestone message in `#milestone-msg`, remove `hidden` class, set a 3 000 ms `setTimeout` to re-add `hidden` and call `MessageDisplay.showIdle()`
    - If called again while timer is running, `clearTimeout` previous timer and reset the 3 000 ms countdown with the new message (Req 6.4 second condition)
    - _Requirements: 6.4, 6.6_

  - [~] 9.3 Implement `DOMContentLoaded` init sequence
    - Cache all DOM references (`#cat-svg`, `#cat-wrapper`, `#message-text`, `#aria-live`, `#mute-toggle`, `#counter-value`, `#milestone-msg`)
    - Run `REACTION_POOL` validation (Task 3.2)
    - Call `MessageDisplay.init()`, `ClickCounter.init()`, `SoundEngine.loadClips()`, `IdleDetector.start()`
    - Verify `#cat-wrapper` has `tabindex="0"` and restore it if missing (Req 9.6)
    - _Requirements: 1.6, 9.6_

- [~] 10. Checkpoint — verify layout and core interaction
  - Open `index.html` in a browser, confirm the cat SVG is centred, the counter reads "0 pets", the mute button shows 🔇, clicking the cat triggers an animation and updates the message, and no console errors appear at load
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 11. Build `test.html` — property-based tests with fast-check
  - [~] 11.1 Create `test.html` scaffold with fast-check CDN
    - Create `test.html` in the workspace root; load fast-check via CDN (`https://cdn.jsdelivr.net/npm/fast-check/lib/bundle/fast-check.min.js`)
    - Copy/import the pure-logic functions under test into `test.html` (or reference them via a `<script src="index.html">` extract — use a shared `logic.js`-style inline approach so functions are testable without a build step)
    - Provide a minimal test runner that logs PASS/FAIL to the console and to a `<pre id="output">` element on the page
    - _Requirements: (testing infrastructure)_

  - [~] 11.2 Implement all 6 property tests (P1–P6)
    - P1 — `ClickCounter.increment` is always exactly +1 (Task 7.2)
    - P2 — No reaction chosen 4+ times consecutively (Task 4.4)
    - P3 — Post-exhaustion first pick differs from last pre-exhaustion pick (Task 4.5)
    - P4 — Duration validation filters out-of-range reactions (Task 3.3)
    - P5 — Milestone detection correct for all non-negative counts (Task 7.3)
    - P6 — MessageDisplay syncs truncated text to both elements (Task 6.3)
    - Each test uses `fc.assert(fc.property(…))` with `numRuns: 100` and the generators defined in the design
    - Tag each test with `// Feature: cat-clicker-app, Property N: <property_text>`
    - _Requirements: (covers 3.1, 4.2, 4.5, 4.6, 6.4, 7.6, 9.4)_

- [ ] 12. Accessibility and responsive polish
  - [~] 12.1 Verify and patch accessibility attributes
    - Confirm `aria-label="Pet the cat"` is present on `#cat-wrapper` at all times (including after reaction)
    - Confirm `aria-live="polite"` region is updated within 100 ms on every reaction trigger (add timing guard if needed)
    - Confirm `#mute-toggle` `aria-pressed` flips correctly on each toggle
    - Confirm focus ring is visible (≥ 2 px outline, ≥ 3:1 contrast) on `#cat-wrapper`
    - _Requirements: 9.1, 9.3, 9.4, 5.3_

  - [~] 12.2 Verify responsive layout at boundary viewports
    - Add a `<meta name="viewport" content="width=device-width, initial-scale=1">` tag if not present
    - Confirm no horizontal scrollbar appears at 320 px, 768 px, 1440 px, and 2560 px using browser DevTools responsive mode
    - Adjust Tailwind classes or `<style>` fallback rules as needed to meet the 320–2560 px requirement
    - _Requirements: 1.4, 2.2, 2.3, 7.4_

  - [ ]* 12.3 Write Playwright integration test for keyboard navigation and aria-live
    - Create `tests/e2e.spec.js`; use Playwright to load `index.html`
    - Tab to `#cat-wrapper`, press Space, assert `#aria-live` contains the reaction message within 100 ms and `#counter-value` equals "1"
    - Press Enter, assert counter equals "2"
    - Click mute toggle, assert `aria-pressed="false"` and icon is 🔊
    - _Requirements: 9.1, 9.2, 9.4, 3.1, 5.3_

- [~] 13. Final checkpoint — full verification
  - Open `test.html` in a browser; confirm all 6 property tests pass (PASS in console/`<pre>` output)
  - Open `index.html`; manually verify: cat is visible, counter increments, idle animation plays after 10 s of inactivity, milestone message appears at 10 pets and is replaced at 25 pets, mute toggle works, keyboard Tab + Space/Enter works, `prefers-reduced-motion` simulation removes animations
  - Ensure all tests pass, ask the user if questions arise.

---

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Property tests are in `test.html` (fast-check via CDN) — no build step required
- All implementation lives in `index.html`; test infrastructure lives in `test.html` (and optionally `tests/e2e.spec.js` for Playwright)
- Each task references specific requirements for full traceability
- Checkpoints (Tasks 10 and 13) are manual verification steps, not automated

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["2.1", "2.2", "3.1"] },
    { "id": 1, "tasks": ["3.2", "4.1", "4.3", "6.1", "6.2", "7.1", "8.1", "8.2"] },
    { "id": 2, "tasks": ["3.3", "4.2", "4.4", "4.5", "5.1", "6.3", "7.2", "7.3"] },
    { "id": 3, "tasks": ["5.2", "9.1", "9.2", "9.3"] },
    { "id": 4, "tasks": ["11.1", "12.1", "12.2"] },
    { "id": 5, "tasks": ["11.2", "12.3"] }
  ]
}
```
