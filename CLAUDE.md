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

---

## Grow-dismiss modal: scrim must lose pointer-events at collapse start

When a modal expand panel closes, the dark scrim behind it (`ngExpandScrim`) still has `pointer-events:auto` while it animates out. It covers the whole phone and absorbs the next card tap, requiring two clicks to open a new card.

Fix: set `pointer-events:none` on the scrim at the very start of `ngCollapseCard`, alongside the panel:

```js
panel.style.pointerEvents = 'none';
document.getElementById('ngExpandScrim').style.pointerEvents = 'none';
ngExpandState = null;
```

The scrim's `pointer-events` is restored to `auto` only when the next modal fully opens (in the expand done block).

---

## Grow transitions: reusable source → panel → source pattern

This should be the default pattern for any grow/shrink transition in this prototype. The popover-chip demo was painful because it looked like a simple variant of the existing grow demos, but it violated one key rule: the collapsed visual layer did not actually contain the collapsed chip's content. The geometry was mostly right, but there was nothing for the chip icon/label to fade back in from.

### 1. The morphing panel owns the solid surface

The panel itself must carry the opaque background for the whole animation:

```html
<div id="tpPopoverPanel" style="background:var(--c-surface-0);overflow:hidden;">
```

Do not put the only white fill on child layers whose opacity crossfades. If both source and destination layers are partially transparent at the same time, the panel becomes see-through over the scrim/background. The source and destination layers should fade their content/stroke, not the container's base fill.

### 2. Use two real visual layers, not "panel border now, source DOM later"

Every grow transition needs:

- **Source visual layer:** looks exactly like the collapsed source while the real source DOM is hidden.
- **Destination visual layer:** full-size expanded content, scaled/clipped by the panel during the morph.
- **Real source DOM:** hidden from expand start until collapse is fully done, then restored.

For cards/search, the source visual is simple: an image skeleton or plain pill is enough. For chips, the source visual must include the chip internals. A blank stroked pill cannot fade in the icon/label.

Popover-chip fix:

```html
<div id="tpPopoverPillLayer">
  <div id="tpPopoverPillStroke"></div>
  <div id="tpPopoverPillChip"></div>
</div>
<div id="tpPopoverContentLayer">
  <div id="tpPopoverInner">...</div>
</div>
```

At expand start, clone the tapped chip's children into the source layer:

```js
function tpSetPopoverPillChip(chipEl) {
  const pillChip = document.getElementById('tpPopoverPillChip');
  pillChip.replaceChildren();
  pillChip.style.opacity = '0';
  if (!chipEl) return;
  chipEl.childNodes.forEach(node => pillChip.appendChild(node.cloneNode(true)));
}
```

Then hide only the actual source element:

```js
if (chipEl) chipEl.style.visibility = 'hidden';
```

Restore that same element only in the collapse done block:

```js
if (chipEl) chipEl.style.visibility = '';
```

Do not hide/restore all chips with `querySelectorAll`; that causes unrelated chips to flash.

### 3. Stroke should be its own rounded layer

Do not use an inset `box-shadow` for a morphing chip stroke. Near the end of the shrink it can look clipped or masked by neighboring fading layers. Use a real border element that inherits the panel radius:

```css
#tpPopoverPillLayer,
#tpPopoverContentLayer {
  border-radius: inherit;
}

#tpPopoverPillLayer {
  z-index: 2;
  pointer-events: none;
  overflow: hidden;
}

#tpPopoverContentLayer {
  z-index: 1;
  overflow: hidden;
}

#tpPopoverPillStroke {
  position: absolute;
  inset: 0;
  box-sizing: border-box;
  border: 1.5px solid var(--c-border-medium);
  border-radius: inherit;
  pointer-events: none;
}
```

The source/chip layer sits above the destination layer so the stroke stays clean during the final shrink frames. `pointer-events:none` keeps the overlay from stealing clicks when the panel is active.

### 4. Crossfade by geometry progress, not only by elapsed time

Elapsed-time fades work for simple card/image grows, but chip popovers need the source layer to appear as the panel gets close to chip geometry. Use shrink progress (`pXW` or `wProg`) and smoothstep ranges:

```js
const contentFade = tpPopoverSmoothstep(pXW, 0.02, 0.34);
const pillFade = tpPopoverSmoothstep(pXW, 0.16, 0.62);
const chipFade = tpPopoverSmoothstep(pXW, 0.28, 0.72);

contentLayer.style.opacity = (1 - contentFade).toFixed(3);
pillLayer.style.opacity = pillFade.toFixed(3);
tpSetPopoverPillChipOpacity(chipFade);
```

Use the same fade logic in gesture/velocity collapse (`tpPopoverCollapseWithVelocity`) with `wProg`. Otherwise tap-dismiss and drag-dismiss diverge.

### 5. Destination content scales inside the clipped panel

Keep expanded content at its full target width and scale it from the top-left as the panel width changes:

```js
inner.style.width = toW + 'px';
inner.style.transform = `scale(${curW / toW})`;     // expand
inner.style.transform = `scale(${curW / fromW})`;   // collapse from current width
```

This prevents reflow while the panel is changing size. The panel clips the scaled content.

### 6. Scrim timing is separate from panel opacity

The scrim can stay dark while the panel is large and fade late in the collapse, but it should not be used to hide missing source visuals. If the source layer is correct, the scrim timing becomes a polish choice instead of a crutch.

```js
scrim.style.opacity = Math.max(0, 1 - Math.max(0, (pXW - 0.6) / 0.4)).toFixed(3);
```

### Source visual layer must contain actual content, not just a background color

The most common failure mode when building a new grow: `pillLayer` / `sourcePillLayer` is an empty div with the right background color. During collapse it fades in as a blank box — visually invisible since it matches the panel background — then the real source element snaps into view at the end.

**Fix:** put a copy of the source element's content inside the source layer:

```html
<!-- WRONG — empty layer, content pops in at end of collapse -->
<div id="tpSearchPillLayer" style="background:var(--c-surface-0);"></div>

<!-- CORRECT — layer contains a visual replica of the collapsed source -->
<div id="tpSearchPillLayer" style="background:var(--c-surface-0); display:flex; align-items:center; justify-content:center;">
  <div class="sk-search-dot"></div>
  <div class="sk-search-text"></div>
</div>
```

For cards, `cardLayer` works because it contains the actual card image element. For chips, `pillLayer` works because it contains cloned chip internals. The same rule applies to any new grow: before thinking the crossfade is broken, check whether the source layer has real content in it.

**Never try to work around this by fading `sourceEl.style.opacity`** — that causes a double-image because the source DOM and the panel are both composited simultaneously. The only correct approach: source layer has content, `sourceEl` is `visibility:hidden` for the full duration.

### Checklist for a new grow transition

1. Measure source rect from the real DOM element.
2. Build a panel with an always-opaque base surface.
3. Add a source visual layer that **contains a replica of the source element's content** (not just a background).
4. Add a destination content layer that can scale without reflow.
5. Hide only the real source element (`visibility:hidden`) from expand start until collapse is fully done.
6. Crossfade source/destination layers while geometry changes.
7. Restore only the real source element in collapse done.
8. Implement the same logic in click collapse and gesture/velocity collapse.
9. Test mid-expand, mid-collapse, near-final collapse, and final state. Do not judge only the settled frame.

---

## Collapse flash when interrupted mid-expand

**Symptom:** The first time you close a zoom transition, it briefly flashes a full-screen frame before playing the collapse animation.

**Cause:** `collapseCard` / `tpCollapseCard` hardcodes the collapse start position to the settled end state (`left:0, top:0, fullWidth, fullHeight`). If the expand animation hasn't fully settled yet, the panel is still mid-flight — so the first collapse frame jumps to full-screen, then immediately starts animating back.

**Fix:** Read the panel's *current* inline style values as the collapse start point instead of assuming settled:

```js
// Instead of:
const fromX = 0, fromY = 0, fromW = phone.clientWidth, fromH = phone.clientHeight, fromR = 40;

// Do:
const fromX = parseFloat(panel.style.left) || 0;
const fromY = parseFloat(panel.style.top)  || 0;
const fromW = parseFloat(panel.style.width)  || phone.clientWidth;
const fromH = parseFloat(panel.style.height) || phone.clientHeight;
const fromR = parseFloat(panel.style.borderRadius) || 40;
```

Apply the same fix to both `collapseCard` (spring zoom demo) and `tpCollapseCard` (transition patterns zoom demo).

---

## Zoom expand: content scale mismatch when phone is inside a CSS scale() container

**Symptom:** The detail screen inside the expand panel appears larger than the same detail screen in a sibling phone. Content density looks different between the two phones.

**Cause (two parts):**

1. `parseFloat(getComputedStyle(phone).getPropertyValue('--phone-scale'))` returns `NaN` when the custom property value is a `calc()` expression. The fallback of `|| 1` silently made the detail content render at 1:1 scale.

2. `getBoundingClientRect()` returns values in scaled viewport pixels. When the phone is inside a container with a CSS `scale()` transform, the card rect deltas must be divided by that scale factor to get phone-frame-local CSS coordinates.

**Fix:**

```js
// Derive scale from clientWidth vs. reference, never parse --phone-scale:
const DETAIL_REF_W = 278;
const phoneContentScale = phone.clientWidth / DETAIL_REF_W;

// Divide getBoundingClientRect deltas by the outer scaler's scale:
const m = scalerEl.style.transform.match(/scale\(([^)]+)\)/);
const scalerScale = m ? parseFloat(m[1]) : 1;
const fromX = (cardRect.left - phoneRect.left) / scalerScale - phone.clientLeft;
const fromY = (cardRect.top  - phoneRect.top)  / scalerScale - phone.clientTop;
const fromW = cardRect.width  / scalerScale;
const fromH = imgRect.height  / scalerScale;
```

**Rule:** Never parse `--phone-scale` with `parseFloat` — CSS `calc()` values are not resolved by `getPropertyValue`. Always derive the scale from `phone.clientWidth / DETAIL_REF_W`.

---

## Zoom jitter: double-fire from phone click handler

**Symptom:** On open, the panel flashes one frame, resets, then plays again.

**Root cause:** A parent `#tpPhone` click listener called `play()` on every click inside the phone. The `closest()` guard failed when the click target was a child of the card — `closest` still matched, but timing races caused expand to fire, set `tpExpandState`, and then the bubbled event triggered `play()` again which immediately called collapse.

**Fix (both required):**

1. Skip `play()` in the phone handler entirely when the active demo is zoom:
```js
document.getElementById('tpPhone').addEventListener('click', (e) => {
  const key = document.querySelector('.tp-tab.active')?.dataset.tp;
  if (key === 'zoom') return;
  TP_DEMOS[key]?.play();
});
```

2. Add `stopPropagation` to card and panel handlers so clicks never reach the phone handler:
```js
card.addEventListener('click', (e) => { e.stopPropagation(); /* ... */ });
document.getElementById('tpExpandPanel').addEventListener('click', (e) => { e.stopPropagation(); /* ... */ });
```

---

## Description text mismatch on page load vs. tab click

**Symptom:** Body copy shows stale/wrong text on initial page load, correct after clicking tabs.

**Cause:** HTML has hardcoded text; JS only updates it on tab click. On clean load with no hash, the initializer doesn't run.

**Fix:**

1. Empty the hardcoded text in HTML: `<p class="section-body" id="tpBodyCopy"></p>`

2. Always call the switch function on load, falling back to the default active tab:
```js
const hashTp = location.hash.match(/^#tp-(.+)$/);
if (hashTp) {
  switchTpDemo(hashTp[1]);
} else {
  const defaultTpKey = document.querySelector('.tp-tab.active')?.dataset.tp;
  if (defaultTpKey) switchTpDemo(defaultTpKey);
}
```

---

## Phone scale system: consistent UI density across phone sizes

**Problem:** Different demo sections use phones of different sizes. Without scaling, the same template content looks denser on smaller phones.

**Solution:** A CSS custom property `--phone-scale` per container, and `.push-dest-inner` / `.phone-content` classes that apply a compensating transform so all content renders at 278px reference density.

```css
#tpDemoContainer .phone-frame  { --phone-scale: 1; }
.phone-frame                   { --phone-scale: calc(298/278); }
#ngDemoContainer .phone-frame  { --phone-scale: calc(318/278); }
#accDemoContainer .phone-frame { --phone-scale: calc(248/278); }

.phone-content, .push-dest-inner {
  transform-origin: top left;
  transform: scale(var(--phone-scale, 1));
  width: calc(100% / var(--phone-scale, 1));
  height: calc(100% / var(--phone-scale, 1));
}
```

**Rules:**
- Home-screen content → stamp into `.phone-content`
- Full-screen destination content → stamp into `.push-dest-inner` inside `.push-screen`
- Never stamp full-screen templates directly into `.push-screen` without the `.push-dest-inner` wrapper

---

## Segmented toggle to swap between two demo variants (e.g. Layouts / Icons)

This pattern lets a single demo tab show two completely different visuals via a segmented button. Took 30+ turns to get right — the failure modes are subtle.

### The three killers

**1. `history.replaceState` throws on `file://` URLs — and silently halts your function.**

If your `applyMode()` function calls `history.replaceState(null, '', '#some-hash')` and the file is opened locally, it throws a security error. Everything after that line — including the DOM visibility swap — never runs. The toggle *appears* broken even though `tpCfMode` is set correctly.

Fix: wrap every `history.replaceState` call in a `try/catch`:

```js
try { history.replaceState(null, '', `#tp-cross-fade-${mode}`); } catch(e) {}
```

**2. `reset()` overwrites `tpCfMode` on every autoplay cycle.**

Autoplay calls `reset()` before each play. If `reset()` contains any line like `tpCfMode = 'cards'` (even guarded by `if (reason !== 'leaving')`), it fires every cycle and reverts whatever the user just selected.

Fix: `reset()` must only reset mode when `reason === 'leaving'`. In all other cases, leave `tpCfMode` untouched.

**3. Calling `applyMode()` from `reset()` — even with the correct mode — causes flicker or re-reverts.**

Even passing `applyMode(tpCfMode)` inside `reset()` was unreliable because `tpCfMode` could be stale at call time depending on execution order.

Fix: Remove `applyMode()` from `reset()` entirely. Instead, `reset()` reads `tpCfMode` and sets layer visibility directly:

```js
reset(reason) {
  if (reason === 'leaving') tpCfMode = 'cards';
  const phone = document.getElementById('tpPhone');
  const iconsDemo = document.getElementById('tpIconsDemo');
  const _icons = tpCfMode === 'icons';
  if (phone) phone.style.display = _icons ? 'none' : '';
  if (iconsDemo) iconsDemo.style.display = _icons ? 'flex' : 'none';
  // ... rest of visual reset ...
}
```

### State variable placement

Declare the mode variable early — alongside `tpAnim`, `tpDemoState`, etc. A `let` declared after the demo object (`TP_DEMOS`) causes a temporal dead zone: `reset()` silently writes to a global instead of the `let`, and the toggle appears broken.

### Showing a layer that has `display:none` in CSS

Use `el.style.display = 'flex'` (or `'block'`), never `el.style.display = ''`. Setting `''` removes the inline style and falls back to the CSS rule, which keeps it hidden.

### Only reset mode on tab leave

Pass a sentinel argument to `reset()` when leaving the tab: `reset('leaving')`. The autoplay scheduler calls `reset()` without arguments — those calls must not touch the mode variable.

---

## Responsive demo container heights

### Transitions section (`#tpDemoContainer`)

At `max-width: 1080px`, the tab bar stacks above the container (tabs move out of the inline flow). A media query adds extra `padding-top` to make room and bumps `min-height` to compensate:

```css
@media (max-width: 1080px) {
  #tpDemoContainer { padding-top: 80px !important; min-height: 708px !important; }
}
```

The base (wide) values are `padding: 32px` and `min-height: 680px` set inline on the element. If the top spacing at narrow widths feels too tight, increase `padding-top` and `min-height` by the same amount.

### Gestures section (`#ngDemoContainer`)

The NG demo uses a JS `ResizeObserver` (`updateNgScale`) that CSS-scales `#ngDemoScaler` down as the container narrows. The container itself is `flex:1` with no fixed height — the row wrapper has `min-height: 776px`. Because the phone shrinks via `scale()` (layout-neutral), it takes less visual space inside a fixed-height container, which naturally creates more padding around it at narrow widths. No extra media query needed; the "grows taller" effect is the fixed container height with a visually shrinking phone inside it.

---

## Demo cross-fade transition (`demoFade` / `demoFadeSwap`)

Every tab click that switches demo content uses a cross-fade: fast ease-out exit, then a spring enter. This is the motion system's own cross-fade pattern applied to itself.

### How it works

**Exit:** CSS transition, `150ms cubic-bezier(0.40,0.00,1.00,1.00)`. Opacity → 0, scale → 0.96, blur → 6px.

**Enter:** Starts 100ms into the exit (overlap). `swap()` is called at the 100ms mark — the element is still fading out but partially invisible. Then a JS RAF spring (stiffness 300, critically damped) drives opacity, scale, and blur back to resting. Opacity ramps over the first 30% of spring travel; scale goes 0.96→1; blur 6→0.

**Dedicated RAF handle:** `_demoFadeRaf` — separate from `tpAnim` so they don't cancel each other.

**`demoFade(el, swap, onReady, { noScale, onProgress })`** — single-element variant. Fades out `el`, calls `swap()` at the 100ms mark, then springs `el` back in. `onProgress(completion)` fires each frame with the 0→1 spring completion value if provided.

**`demoFadeSwap(outEl, inEl, swap)`** — two-element variant for spring demos where the outgoing and incoming elements are different DOM nodes (e.g. switching between spring demo panels while keeping the bezel static). Fades out `outEl`, resets it, primes `inEl` at opacity 0 / scale 0.96, calls `swap()`, then springs `inEl` in.

### What each section targets

| Section | Element faded | Why |
|---------|--------------|-----|
| Transition Patterns (TP) | `#tpPhoneScreen` (content wrapper inside bezel) | Keeps bezel static |
| TP → slide-up-fade (device change) | `#tpDeviceWrap` (phone+desktop+icons container) | Device type changes, whole device fades |
| Gestures / NG | `#ngPhoneScreen` (content wrapper inside bezel) | Keeps bezel static |
| Bezier | `#dtContent` (inside `#dtScaler`) | `dtScaler` has its own `transform:scale()` from ResizeObserver — targeting the inner wrapper avoids overwriting it |
| Spring | `.spring-screen` wrappers (JS-created) via `demoFadeSwap` | Bezel stays put; two different panels swap |

### Adding a new section

1. Create a content wrapper (`position:absolute;inset:0;overflow:hidden`) inside the device bezel.
2. Target that wrapper in your tab click handler — never the outer container or the bezel itself.
3. Use `demoFade(wrapper, () => switchMyDemo(key), onReady)` for a single phone/device.
4. Use `demoFadeSwap(outScreen, inScreen, swap)` if the outgoing and incoming content are different DOM elements.
5. If your device type changes on some tab switches (like TP's slide-up-fade), detect that case and target a wider wrapper instead.

### NG sheet-dismiss: black background aliasing fix

**Symptom:** When switching to the sheet-dismiss tab, thin white lines flash at the edges of the phone as the spring scale-up finishes.

**Root cause:** `#ngPhoneScreen` scales to 0.96 during the spring enter, pulling away from the edges of `#ngPhone`. `.phone-frame` has `background: var(--c-surface-0)` (white/light), which shows through the gap at the corners and edges as the scale resolves.

**Fix — three parts:**

1. **`border-radius` on `#ngPhoneScreen`:** The phone frame is `border-radius: 46px` with a `6px` border, so inner content radius is `40px`. Add `border-radius:40px` to `#ngPhoneScreen` so its corners match during scale — eliminates corner shape mismatch.

2. **Black background on `#ngPhone`:** Set `#ngPhone.style.background = '#000'` to fill the gap. This must be scoped to sheet-dismiss only:
   - **Entering sheet-dismiss:** set it inside the `swap()` callback (element is at opacity 0 — invisible). Use a `200ms linear` CSS transition with a `200ms` delay so it fades in late rather than snapping, keeping it invisible during the early part of the spring:
     ```js
     ngPhone.style.background = 'transparent';
     ngPhone.style.transition = '';
     requestAnimationFrame(() => {
       ngPhone.style.transition = 'background 200ms linear 200ms';
       ngPhone.style.background = '#000';
     });
     ```
   - **Leaving sheet-dismiss:** clear background and transition immediately *before* `demoFade` starts, so the exit fade plays clean with no black:
     ```js
     if (prevKey === 'sheet-dismiss') {
       ngPhone.style.transition = '';
       ngPhone.style.background = '';
     }
     ```

3. **Why delay the fade-in:** setting the black instantly in `swap()` causes a visible black flash at the start of the spring enter, even though `#ngPhoneScreen` is fading in. The 200ms delay + 200ms fade means the black only becomes visible late in the animation — right when the white gaps would otherwise appear — and it reads as part of the content resolving rather than a separate background pop.

---

## Play/pause system for multi-step demos (e.g. grow-sheet)

The autoplay loop runs via `startTp` / `stopTp` and a `scheduleNext(ms, fn)` timer chain. Understanding the architecture is critical when adding a new multi-step sequence.

### How the loop works

```
startTp() → scheduleNext(300, loop) → loop() → runCycle(key, loop)
```

`runCycle` fires the animations in sequence using nested `scheduleNext` callbacks. Each step waits for the previous animation to complete (via `onDone` callback) then schedules the next one. `tpPlaying` is checked at every callback entry — if `stopTp()` was called, the chain halts.

### stopTp / startTp contract

`stopTp` must:
1. Set `tpPlaying = false`
2. Cancel `tpLoopTimer` (the next scheduled step)
3. Cancel any in-flight `rAF` animation handles (e.g. `tpModalFullAnim`)
4. Cancel `tpScalePulse` if running
5. Snap all panels to the nearest valid settled state

`startTp` with `{ resume: true }` must detect which settled state the demo is in and continue from the correct next step — NOT call `reset()` which wipes all state.

### Multi-step demos need phase tracking

For sequences where the same visual state (e.g. "sheet visible") can occur at two different points in the sequence, a phase flag is required to resume correctly.

Example: `tpModalSheetPhase = 'pre' | 'post'`
- Set to `'pre'` when card expands to sheet (not yet gone fullscreen)
- Set to `'post'` when fullscreen collapses back to sheet
- Resume logic checks this flag to know whether to expand-to-full (`'pre'`) or collapse-card (`'post'`)

Without this flag, pausing on the sheet post-fullscreen would incorrectly resume by re-expanding to full.

### Snap-on-pause logic for mid-animation states

When `stopTp` fires while `tpModalFullAnim` is running (mid-spring), check `tpModalFullState` to determine direction:
- `tpModalFullState !== null` → mid expand-to-full → snap to fullscreen settled state
- `tpModalFullState === null` → mid collapse-to-sheet → snap to sheet settled state using `tpExpandState.toX/Y/W/H/R`

For the fullscreen settled state (animation complete, `tpModalFullAnim` null, `tpModalFullState` set) — leave CSS as-is. Resume → collapse to sheet.

For the sheet settled state (`tpExpandState` set, `tpModalFullState` null) — leave CSS as-is. Resume → check `tpModalSheetPhase`.

### Demo tab switch leaves panel visible (recurring bug)

**Symptom:** Switch away from a multi-step demo mid-animation, come back, and the panel is stuck on screen — wrong opacity, wrong position, or two close buttons visible.

**Cause:** `reset()` is called on the outgoing demo when switching tabs, but if any of these are true the panel stays broken:
- An in-flight `rAF` handle (e.g. `tpModalFullAnim`) is still running after `reset()` — it fires one more frame and overwrites the reset
- `tpModalFullState` / phase flags are not cleared, so the next autoplay start resumes mid-sequence instead of from scratch
- Layer opacities (e.g. `tpModalWebsiteLayer`) are not reset, leaving ghost content visible

**Fix — `reset()` must do all of these for every animation handle it owns:**

```js
reset() {
  // 1. Cancel every rAF handle
  if (tpExpandAnim)    { cancelAnimationFrame(tpExpandAnim);    tpExpandAnim    = null; }
  if (tpModalFullAnim) { cancelAnimationFrame(tpModalFullAnim); tpModalFullAnim = null; }
  tpScalePulseCancel();

  // 2. Clear all state flags
  tpModalFullState  = null;
  tpModalSheetPhase = 'pre';
  tpExpandState     = null;

  // 3. Hide every panel and reset every layer opacity
  modPanel.style.display  = 'none';
  modPanel.style.transform = '';
  document.getElementById('tpModalContentLayer').style.opacity  = '0';
  document.getElementById('tpModalWebsiteLayer').style.opacity  = '0';
  document.getElementById('tpModalWebsiteLayer').style.pointerEvents = 'none';
  // ... etc for every layer that the animation touches
}
```

**Rule:** For every CSS property an animation writes to, `reset()` must write the settled/hidden value. If you add a new layer or property to an animation, add it to `reset()` in the same PR.

---

### Adding a new multi-step sequence

1. Add an `onDone` callback parameter to every animation function in the chain
2. In `runCycle`, nest `scheduleNext` calls inside those `onDone` callbacks
3. Check `if (!tpPlaying) return;` at the start of every `scheduleNext` callback
4. In `stopTp`, cancel all rAF handles for the new sequence and snap to the nearest settled state
5. In `startTp` resume path, detect which settled state you're in and continue from the right step
6. If the same visual state appears multiple times in the sequence, add a phase flag

---

## Stamping a template into a flying panel (grow/expand destination)

When a grow transition expands to reveal a full-screen template (not a `.push-screen` or `.phone-content`), the inner content wrapper needs explicit width + scale management. **Do not use `.push-dest-inner` class** — it uses `calc(100% / var(--phone-scale))` which doesn't work inside a JS-sized panel.

**The correct pattern** (matches `tpDetailInner`, `ngDetailInner`):

```html
<!-- Fixed 278px reference width, transform-origin top left, JS drives the scale -->
<div id="myInner" style="position:absolute;top:0;left:0;width:278px;transform-origin:top left;display:flex;flex-direction:column;gap:5px;padding:14px 18px 18px;box-sizing:border-box;">
</div>
```

```js
const REF_W = 278;
// On init (before animation):
inner.style.width = REF_W + 'px';
inner.style.transform = `scale(${panelTargetWidth / REF_W})`;

// Every animation tick as panel width changes:
inner.style.transform = `scale(${curPanelWidth / REF_W})`;
```

**Why:** The panel's own width is set by JS during the spring animation. A percentage-based inner div will reflow on every frame. A fixed 278px div scaled to `curW / 278` gives smooth, reflow-free content that matches the reference density of every other phone template.

**Padding:** Use `padding: 14px 18px 18px; box-sizing: border-box` for full-screen destinations (small top for status bar area). The navbar (`.ws-navbar`) uses `margin: 0 -18px` to break out to full width since it has its own internal padding.

**Where `.push-dest-inner` IS correct:** inside `.push-screen` containers (push transitions, room detail). Those containers have a stable CSS width so `calc(100% / var(--phone-scale))` resolves correctly.

---

## Inset modal sheet height measurement (`tpGetModalDest` / `ngGetModalDest`)

The sheet height is measured dynamically each time a card is tapped — it's not cached. The panel is briefly shown off-screen (`visibility:hidden`) and its height is read.

**The bug:** `panel.scrollHeight` always returns 0 (or near-0) because `tpModalInner`/`ngModalInner` is `position:absolute` inside the panel. Absolutely-positioned children don't contribute to a parent's `scrollHeight`.

**The fix:** Measure `inner.scrollHeight` instead:

```js
if (inner) inner.style.transform = 'scale(1)';
const innerH = inner ? inner.scrollHeight : 0;
const toH = (innerH > 0 ? innerH : panel.scrollHeight) || 320;
```

`inner.scrollHeight` captures the true stacked height of all the content (image + text + CTA + padding). The `panel.scrollHeight` fallback covers the case where inner is null, and `|| 320` is the last-resort default.

---

## `springPos` return value direction — the trap that kills inset-sheet animations

`springPos(t, stiffness, damping)` returns **1 at t=0 and decays to 0**. This is the opposite of what you might expect (0→1 "progress"). Getting this wrong produces animations that flash or reverse.

**Correct formulas for slide-up/slide-down:**

```js
// Opening: panel slides from 110% (off-screen) → 0% (visible)
// p starts at 1 → decays to 0, so p*110 starts at 110 → 0 ✓
insetPanel.style.transform = `translateY(${p * 110}%)`;
insetScrim.style.opacity   = String(1 - sp);  // sp: 1→0, so 1-sp: 0→1 (fades in) ✓

// Closing: panel slides from 0% → 110%
// (1-p) starts at 0 → goes to 1, so (1-p)*110: 0→110 ✓
insetPanel.style.transform = `translateY(${(1 - p) * 110}%)`;
insetScrim.style.opacity   = String(sp);       // sp: 1→0 (fades out) ✓
```

**Mental model:** think of `p` as "how far from the destination you still are" — 1 = just started, 0 = arrived.

**Always use `tpSpringAnimate`** for these, not a custom `requestAnimationFrame` tick. The custom tick shares `tpAnim` with `tpSpringAnimate`, so they conflict unpredictably. `tpSpringAnimate` is the single rAF manager — use it everywhere.

**Why this matters for content changes:** Any time you change the inset modal template (bigger image, more padding, extra rows), the sheet will auto-resize correctly on next open. You never need to hardcode a height.

---

## Press-to-scale + drag-to-dismiss + X-to-dismiss pattern

This is the inset sheet interaction pattern. Reuse it for any floating element that should feel physically responsive to touch.

### The two-element split (critical)

The slide animation (`translateY`) and the scale animation must live on **separate elements** or they'll fight each other:

```html
<!-- Outer: owns translateY for slide/drag -->
<div id="mySlide" data-open="false" style="position:absolute; transform:translateY(110%);">
  <!-- Inner: owns scale for press/pulse -->
  <div id="myPanel" style="transform-origin:center center;">
    <!-- content -->
  </div>
</div>
```

Never put both on the same element.

### State variables (must be outer scope, not inside an IIFE)

`play()` and the drag/press handlers live in different scopes. If you declare these inside an IIFE they'll be invisible to `play()` and cancel will silently fail:

```js
let tpInsetAnimating  = false;  // blocks re-entry during animation
let tpInsetPressRaf   = null;   // tracks the press spring RAF
let tpInsetPressScale = 1.0;    // tracks current scale so springs chain correctly
```

### Press spring

On `pointerdown`, spring the inner element toward 0.96. Use a **critically damped** ratio (1.0) for press-down so it eases straight in with no overshoot:

```js
container.addEventListener('pointerdown', (e) => {
  if (!e.target.closest('#mySlide')) return;
  animatePressScale(pulse, 0.96, 350, 1.0); // critically damped press-down
  dragActive = true;
  dragStartY = e.clientY;
}, { passive: true });
```

On `pointerup`, spring back with bounce (ratio ~0.50):

```js
container.addEventListener('pointerup', (e) => {
  if (e.target.closest('.my-close-btn')) return; // X will dismiss — don't spring back
  const dy = Math.max(0, e.clientY - dragStartY);
  if (dy > 4) return; // drag handler takes over
  animatePressScale(pulse, 1.0, 300, 0.50);
}, { passive: true });
```

The `animatePressScale` function springs from the **current** `tpInsetPressScale` to the target, so chains work correctly mid-animation:

```js
function animatePressScale(pulse, toScale, k, ratio) {
  if (tpInsetPressRaf) { cancelAnimationFrame(tpInsetPressRaf); tpInsetPressRaf = null; }
  const d = calcDamping(k, ratio);
  const from = tpInsetPressScale;
  const startT = performance.now();
  function tick(now) {
    const t = (now - startT) / 1000;
    const p = springPos(t, k, d);
    tpInsetPressScale = from + (toScale - from) * (1 - p);
    pulse.style.transform = `scale(${tpInsetPressScale.toFixed(4)})`;
    if (t < 3 && !isSettled(t, k, d)) { tpInsetPressRaf = requestAnimationFrame(tick); }
    else { tpInsetPressRaf = null; tpInsetPressScale = toScale; pulse.style.transform = toScale === 1 ? '' : `scale(${toScale})`; }
  }
  tpInsetPressRaf = requestAnimationFrame(tick);
}
```

### Drag

While dragging, lock the scale where it is (don't reset it):

```js
document.addEventListener('pointermove', (e) => {
  if (!dragActive) return;
  const dy = Math.max(0, e.clientY - dragStartY);
  if (dy > 4) {
    // Lock scale — cancel RAF but leave transform as-is
    if (tpInsetPressRaf) { cancelAnimationFrame(tpInsetPressRaf); tpInsetPressRaf = null; }
  }
  slide.style.transform = `translateY(${dy}px)`;
}, { passive: true });
```

On drag release: if past threshold → animate off screen; if not → spring both translateY and scale back:

```js
} else if (dy > 4) {
  tpSpringAnimate(...spring slide back...);
  animatePressScale(pulse, 1.0, 300, 0.50); // spring scale back simultaneously
}
```

### X-to-dismiss

When `play()` is called to close, cancel ALL scale animations before the dismiss slide starts — both the press spring and any `tpScalePulse`:

```js
tpScalePulseCancel();
if (tpInsetPressRaf) { cancelAnimationFrame(tpInsetPressRaf); tpInsetPressRaf = null; }
tpInsetPressScale = 1.0;
// Do NOT clear insetPanel.style.transform here — leave scale at wherever it is
// The slide-down plays over whatever scale state the panel is at
setTimeout(startSlide, 80); // small delay so scale-down is visible before slide
```

**The trap:** clearing `insetPanel.style.transform = ''` before the slide starts causes a snap to 100% scale. Leave the transform alone — the slide-down animation runs on the outer element, so the inner scale just stays put.

### Done callback cleanup

Always reset panel scale when close finishes:

```js
}, TP_BOUNCE_SPRING, () => {
  slide.style.transform = 'translateY(110%)';
  slide.dataset.open = 'false';
  if (isOpen) { tpScalePulseCancel(); panel.style.transition = ''; panel.style.transform = ''; }
  tpInsetAnimating = false;
  if (onDoneCallback) onDoneCallback();
});
```

### Scrim dismiss

The scrim has `pointer-events:none` so clicks fall through to the container. Check if the click landed outside the panel instead:

```js
if (!e.target.closest('#mySlide')) { triggerDismiss(); return; }
```

---

## Gesture dismiss demos — two known bugs and their fixes

These notes apply to `#gestures-fullscreen-dismiss` and `#gestures-popover-dismiss`.

### Bug 1: Corner rounding snaps on dismiss (wrong `fromR`)

**Symptom:** When a card collapses back to its placeholder after a dismiss gesture, the panel's corner radius snaps — it settles at one value, then the placeholder card appears at a different radius.

**Root cause:** `ngExpandCard` and `tpExpandCard` hardcoded `fromR = 10` (or `fromR = 14`) as a constant instead of reading the actual CSS computed value. The card elements have `border-radius: 14px` in CSS, so when the panel settled at 10px and then the card reappeared at 14px, you saw a snap.

**Fix — read computed style at expand time:**
```js
// ngExpandCard:
const fromR = imgEl ? parseFloat(window.getComputedStyle(imgEl).borderRadius) || 14 : 14;

// tpExpandCard (or wherever fromR is computed):
const fromR = cardEl ? parseFloat(window.getComputedStyle(cardEl).borderRadius) || 14 : 14;
```

**Fix — corner rounding during gesture drag:** During gesture-based dismiss, the code was interpolating `cornerR = toR + (40 - toR) * progress`, hardcoding 40 as the "source" radius. Replace 40 with the saved `fromR`:
```js
// ng drag handler:
const targetCornerR = ngExpandState?.fromR ?? 10;
const cornerR = toR + (targetCornerR - toR) * progress;

// tp drag handler:
panel.style.borderRadius = (toR + ((tpExpandState?.fromR ?? 14) - toR) * progress) + 'px';
```

### Bug 2: Panel snap at end of first dismiss (stale card position)

**Symptom:** When dismissing a card (especially the first time), the panel springs correctly but snaps at the very end — the panel briefly clips or appears in the wrong position before the card reappears.

**Root cause:** `toX/toY` (the collapse destination) is taken from `ngExpandState.fromX/fromY`, which was captured at *expand time*. If the phone layout reflowed between expand and collapse (e.g. first-paint layout settling, or the panel being shown for the first time causes a reflow), the card's rect shifts by a few pixels. The panel springs to the stale expand-time position, then the card reappears at its current position — a visible snap.

This is confirmed via logging: `panel settled at 23.8, 214.2` but `card rect NOW: 20.6, 211.6` — 3px difference.

**Fix — re-measure card rect at collapse start:**

```js
// ngCollapseCard and tpPopoverCollapseWithVelocity:
let toX = savedFromX, toY = savedFromY; // fallback to expand-time
if (card) {
  const phoneRect = phone.getBoundingClientRect();
  const cardRect = card.getBoundingClientRect();
  toX = cardRect.left - phoneRect.left - phone.clientLeft;
  toY = cardRect.top  - phoneRect.top  - phone.clientTop;
}
```

**Rule:** Never use `fromX/fromY` from expand state as the collapse target. Always re-measure via `getBoundingClientRect()` at collapse time. The card is `visibility:hidden` but still in layout — its rect is accurate.

### Also: clamp left/top in all collapse ticks

**Symptom:** Brief edge clipping during the spring animation on near-zero-position cards.

**Root cause:** Spring physics overshoot pushes `simX.x` / `simY.x` briefly below 0. Clipped by `phone-frame`'s `overflow:hidden`.

**Do NOT fix with `overflow:visible` on the phone frame** — that leaks all phone content outside the device border (major regression).

**Fix — clamp in every collapse tick:**
```js
panel.style.left = Math.max(0, simX.x) + 'px';
panel.style.top  = Math.max(0, simY.x) + 'px';
```
Apply to: `tpCollapseCard` (both ticks), `ngCollapseCard`, `ngRevertToExpanded`, `tpPopoverCollapseWithVelocity` (both ticks), `tpPopoverRevertToExpanded`, `tpSearchCollapseWithVelocity` (both ticks).
