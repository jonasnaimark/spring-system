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
