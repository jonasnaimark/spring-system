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
