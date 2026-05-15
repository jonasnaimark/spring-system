# Motion System — Dev Notes

Project-specific patterns and gotchas for working on this file.

---

## Preventing tab label width jump on bold-active

Any time a tab/chip/pill bolds its label when selected, the wider text shifts the layout. Fix with a CSS-only pseudo-element that permanently reserves the bold width:

```css
.your-tab {
  display: inline-flex;
  flex-direction: column; /* stack ::after below the label */
  align-items: center;
}

/* Renders the label at bold weight but invisible, holding the wider size at all times */
.your-tab::after {
  content: attr(data-label);
  font-weight: 500; /* match your active font-weight */
  height: 0;
  overflow: hidden;
  visibility: hidden;
  pointer-events: none;
}
```

Then set `data-label` on each tab from its text content in JS:

```js
document.querySelectorAll('.your-tab').forEach(tab => {
  tab.dataset.label = tab.textContent.trim();
});
```

**Why `flex-direction: column`:** without it, `::after` sits beside the label text as a flex sibling, doubling the width. Column stacks it below at zero height.

---

## Stacked sheet demo — what it took to get smooth

This was a 30+ prompt slog. Document the hard-won lessons.

### 1. Step counter, not toggle

The demo uses a `tpSlideUpStep` integer (0–5) stepped forward by `play()` and backward by drag/click dismiss. The autoplay loop must use a custom path (`slide-up` branch in `runCycle`) that fires each step sequentially — NOT the standard enter/exit toggle which calls `play()` twice. The toggle approach completely breaks the step sequence.

### 2. `snapToStep(n)` as the authoritative state reset

After any dismiss animation completes, call `snapToStep(tpSlideUpStep)` to snap every sheet to its exact correct position/opacity/overlay. Do NOT try to set individual properties inline — one missed property (e.g. `s0.style.opacity`) causes invisible sheets or flashes. `snapToStep` is the single source of truth.

### 3. Spring overshoot causes jump on click-dismiss

The spring animation leaves the sheet slightly above resting position (negative translateY from overshoot). When a click-dismiss fires:
1. Cancel `tpAnim` immediately: `if (tpAnim) { cancelAnimationFrame(tpAnim); tpAnim = null; }`
2. Snap to `translateY(0px)` explicitly
3. Wait **two rAF frames** before starting dismiss animation — one frame is not enough because the cancelled spring's final rAF callback may still fire before yours

```js
sheet.style.transform = 'translateY(0px)';
requestAnimationFrame(() => requestAnimationFrame(() => {
  animateDismiss(sheet, below, 0, noBelow);
}));
```

### 4. Read actual rendered position via DOMMatrix

When starting dismiss from a drag or mid-spring position, use `DOMMatrix` not regex on `style.transform` — the spring uses `%` units, regex only catches `px`:

```js
function parseTY(el) {
  return new DOMMatrix(getComputedStyle(el).transform).m42;
}
function parseSC(el) {
  return new DOMMatrix(getComputedStyle(el).transform).a;
}
```

Capture below-sheet state at **drag end** (pointerup), not drag start — the sheet has moved by then.

### 5. Drag uses document-level listeners, not container

`pointermove`/`pointerup` on the container miss events when the pointer moves outside. Put them on `document`. Do NOT cancel the spring on `pointerdown` — cancel it on the first `pointermove` instead. Cancelling on `pointerdown` also clears `tpSheetAnimating`, which lets the subsequent `click` event slip through.

### 6. 3→2 transition: s0 must fade in

When dismissing from 3 sheets to 2, `s0` has `opacity:0` from the forward animation. `animateDismiss` must explicitly animate it back to 1 in sync:

```js
const s0FadeIn = tpSlideUpStep === 3;
// ...in the frame loop:
if (s0El) s0El.style.opacity = String(ease);
```

### 7. Spam click guard — two-layer approach

**Layer 1: `tpSheetAnimating` flag.** Set `true` at the start of `play()` and `animateDismiss()`. Clear in `onDone` AND in `reset()`. Check this first in every click handler before anything else — even before checking `tpPlaying`. If the flag is stuck (spring cancelled mid-way), a safety `setTimeout` of 600–800ms clears it.

**Layer 2: `tpSheetClickCooldown` timestamp.** After `onDone` fires, set `tpSheetClickCooldown = performance.now() + 150`. Block clicks until `performance.now() > tpSheetClickCooldown`. 150ms is enough to absorb a double-tap without feeling sluggish.

**Critical gotcha:** `softStop()` (cancel loop, set `tpPlaying=false`) must only fire when `tpSheetAnimating` is false. If it fires mid-spring, the spring's `onDone` still runs, but no next step is scheduled — the demo halts mid-sequence. Always check `tpSheetAnimating` before `tpPlaying` in handlers.

**Soft stop vs hard stop:** clicking during autoplay should cancel the *scheduler* (loop timer) but let the current spring finish. Use `softStop()` — cancel the `setTimeout` loop but leave `tpAnim` running. The spring completes, fires `onDone`, clears the flag, and the UI is in a clean state.

```js
function softStop() {
  tpPlaying = false;
  cancelLoop(); // clears tpLoopTimer only
  // does NOT cancel tpAnim — spring runs to completion
}
```

### 8. Pointer events on the scrim

`.push-scrim` has `pointer-events: none` globally. The sheet-scrim needs `pointer-events: auto` inline to receive clicks. Give it its own listener that only fires when `tpSlideUpStep > 0` — otherwise let the click fall through to the container so the empty-state "open first sheet" path works.

**Why not `min-width` in JS:** measuring bold width at runtime is font-timing dependent — if the custom font hasn't loaded yet the measurement uses the fallback font and is wrong.

---

## Demo alt pattern (segmented toggle showing different versions of a demo)

See the comment block in `index.html` above the zoom mode toggle — it has a full numbered checklist. Key gotchas:

1. **Declare mode state variable early** — alongside `tpAnim`, `tpDemoState` etc. at the top of the TP state block. A `let` declared after `TP_DEMOS` causes a temporal dead zone: `reset()` silently writes to a global instead of the `let`, so the toggle appears broken.

2. **Use `'block'` not `''`** when showing a layer that has `display:none` in a CSS rule. Setting `el.style.display = ''` removes the inline style and falls back to the CSS rule, keeping it hidden.

3. **`reset()` must not wipe mode** — autoplay calls `reset()` on every start. If `reset()` resets the mode variable, it undoes any mode the user just selected. Only reset mode when leaving the tab, using a sentinel argument: `reset('leaving')`.

4. **Update the URL** in your `applyMode()` function: `history.replaceState(null, '', \`#tp-{tab}-{mode-slug}\`)`. Also handle the hash at page load alongside the existing zoom-modal case.

---

## Three-sheet pool (NG sheet-dismiss demo) — what it took to get smooth

The "sheet dismiss" demo uses three DOM elements rotating through top/below/spare roles. Getting this right took 20+ iterations. Document the hard-won lessons.

### 1. Three-element pool with rotating indices

`ngPool[ngT]` = top (z:12), `ngPool[ngB]` = below (z:11), `ngPool[ngS]` = spare (z:9/10). After each open or dismiss, the indices rotate:
- Open: `ngT=oldS, ngB=oldT, ngS=oldB`
- Dismiss: `ngT=oldB, ngB=oldS, ngS=oldT`

`ngPlaceTop`, `ngPlaceBelow`, `ngHideSpare` are the single source of truth for resting state. Never set individual properties inline when the element is at rest — always call one of these.

### 2. Never call ngStamp during an active pointer/gesture

`ngStamp` clears and rebuilds the sheet's innerHTML. If called while a `pointerdown` is active (even before `pointermove`), the DOM mutation drops pointer capture and the gesture gets stuck. Only stamp during click events or animation done callbacks — never in `pointerdown`.

### 3. Spare starts 6px below resting, not off-screen

Initial attempt: spare started at `translateY(100%)` and slid in during drag — it only became visible in the last few pixels because the sheet is ~380px tall. Fix: spare lives at `NG_PUSHED_Y + NG_SPARE_OFFSET` (just 6px below its resting spot) at z:9. On `pointerdown` it surfaces to z:10 at that same offset. During drag it slides the 6px to `NG_PUSHED_Y`. Subtle but immediately visible.

### 4. Capture spare's TY at pointerup, not pointerdown

If you capture the spare's start position at `pointerdown` and the user drags, the spare has moved by `pointerup`. `ngDismiss` must capture `ngParseTY(spare)` at `pointerup` time so it animates from wherever the spare currently is, not where it started.

### 5. Spare must use the same scale as ngPlaceBelow throughout

Early version used a separate `NG_SC2 = 0.88` for the spare. When the dismiss animation landed and `ngPlaceBelow` snapped it to `NG_SC1 = 0.94`, there was a visible scale jump. Fix: use only `NG_SC1` for the spare at all times.

### 6. ngSpringBack must set ngSheetAnimating = true

`ngSpringBack` runs when the user drags partway and releases below the dismiss threshold. If `ngSheetAnimating` is not set, a fast tap during the spring-back fires `ngOpen` concurrently. Both animation loops write to the same element, leaving the `.sheet-overlay` opacity stuck at a partial value — the sheet appears tinted by its own scrim. Fix: set `ngSheetAnimating = true` at the start of `ngSpringBack`, clear it in its done callback.

### 7. Skip spring-back entirely on clean taps (dy ≤ 4px)

After adding the animation lock to `ngSpringBack`, clicks stopped working: `pointerdown` showed the spare, `pointerup` fired `ngSpringBack` (dy=0), which locked `ngSheetAnimating`, blocking the subsequent `click` event from reaching `ngOpen`. Fix: in `pointerup`, only call `ngSpringBack` if `dy > 4`. For clean taps, call `ngHideSpare` directly and let the click through.

### 8. Use DOMMatrix to read computed transform, not regex

The spring animation sets transforms in `%` units (e.g. `translateY(45%)`). Regex on `style.transform` only catches `px`. Always use:

```js
function ngParseTY(el) { return new DOMMatrix(getComputedStyle(el).transform).m42; }
function ngParseSC(el) { return new DOMMatrix(getComputedStyle(el).transform).a; }
```

### 9. Clamp spring overshoot in ngOpen

`springPos` returns values slightly below 0 during overshoot (underdamped spring). In `ngOpen`, `c = 1 - p` can briefly exceed 1, pushing the `.sheet-overlay` opacity past 0.3 and the transform past `NG_PUSHED_Y`. When the animation ends and `ngPlaceBelow` snaps to the correct value, there's a visible pop. Fix:

```js
const c = Math.max(0, Math.min(1, 1 - p));
```

### 10. ngSheetScrim must be below z:10

The background scrim (`ngSheetScrim`) was at z:10 — same as the spare sheet. It covered the spare. Fix: set `z-index:8` on `ngSheetScrim` in the HTML.

---

## Animating a flex pill from full-width to fitted (e.g. CTA bar → Back/Next bar)

When a `flex:1` pill needs to shrink to a fixed width (e.g. from a full-width CTA button to a 64px pill), **never animate the pill's own width**. Mixing explicit `width` with `flex:1` causes layout jumps and snapping.

**The correct approach:** animate a sibling element's width instead, and let flex respond naturally.

```js
// Back wrapper starts at 0, grows to push the CTA pill down to target size
const PILL_W = 64;
const BACK_W = bar.clientWidth - 32 - PILL_W; // bar padding is 16px each side

// In animation tick:
back.style.width = (startBackW + (endBackW - startBackW) * completion) + 'px';
// CTA is flex:1 — never set its width. It responds automatically.
```

**Why:** `flex:1` and explicit pixel widths fight each other. Animating a sibling and letting flex fill the remainder is stable, smooth, and requires no cleanup.

**Measuring the sibling target width:** don't use the icon's visual size — compute `barInnerWidth - targetPillWidth`. This ensures the pill lands at exactly the right size regardless of bar dimensions.
