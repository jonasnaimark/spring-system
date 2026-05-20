# Popover Collapse Bug — Debug Log

## What the transition should do

Chips in `#tpPhoneContentPopover` tap to grow into a centered popover overlay. On dismiss:

1. Panel springs back to the chip's exact position/size
2. Popover content (`contentLayer`) fades out over ~200ms as panel shrinks
3. Chip content fades in as panel shrinks (not snap in at the end)
4. Box shadow fades out during shrink
5. Scrim fades out — but stays dark while panel is large, only clearing as panel nears chip size
6. No chip flashing or disappearing

---

## Key DOM structure

```html
<div id="tpPopoverScrim" style="position:absolute;inset:0;background:rgba(0,0,0,0.32);opacity:0;z-index:15;" />
<div id="tpPopoverPanel" style="display:none;position:absolute;z-index:16;overflow:hidden;">
  <div id="tpPopoverPillLayer">
    <!-- opaque surface-0 background + inset stroke only. NO chip content. -->
  </div>
  <div id="tpPopoverContentLayer">
    <div id="tpPopoverInner">
      <!-- full popover content -->
    </div>
  </div>
</div>
```

**Chips** (`.tp-popover-chip`): `background: transparent`, `border: 1.5px solid var(--c-border-medium)`, contain icon + label. They sit in the phone content layer behind the panel.

**Inner div pattern**: `width` is fixed to `toW` (full popover width). During shrink, `transform: scale(curW / toW)` makes it appear to shrink with the panel (since `overflow:hidden` clips it). Origin is top-left.

---

## The core problem: pillLayer ≠ chip visual

The `pillLayer` is just an opaque white rect with a stroke. It does NOT contain the chip's icon and label text. So as the panel shrinks, the content that should be "fading in" (icon + label bars) is never visible inside the panel — it only becomes visible when the actual DOM chip element shows through at the end.

The `chipEl` (the real DOM chip) has `visibility: hidden` set at expand time (line 5574 of `tpExpandPopover`) so it doesn't show through the scrim during the open animation. The restore `chipEl.style.visibility = ''` must happen at the **end** of collapse.

---

## What's been tried

### Attempt 1 — Hide all chips with querySelectorAll, restore at end
**Code:** Set `visibility:hidden` on all `.tp-popover-chip` at collapse start; restore all at collapse end.
**Result:** All 3 chips flash out and back in on every dismiss. Visually jarring.

### Attempt 2 — Restore chipEl visibility at start of collapse (before tick)
**Code:** `chipEl.style.visibility = ''` immediately before the RAF loop starts.
**Result:** "This is worse" — chip snaps back to its background position while panel is still large and scrim is fading, so you see the chip at its resting position underneath the shrinking panel.

### Attempt 3 — Tie scrim fade to panel spring progress; remove all chip hiding
**Code:** 
- Removed all `visibility:hidden` / `querySelectorAll` from both collapse functions
- Changed scrim: `Math.max(0, 1 - Math.max(0, (pXW - 0.6) / 0.4))` — stays at 1 until panel is 60% shrunk, then fades over the final 40%
- Also removed `simScrim` SpringSim from `tpPopoverCollapseWithVelocity`

**Result:** Chips disappear after each shrink (not visible between cycles). Root cause: `chipEl.style.visibility = ''` was previously in the `querySelectorAll` restore loop at collapse end. When the querySelectorAll was removed, the restore was also removed. But `chipEl.style.visibility = 'hidden'` is still set during expand (line 5574). Without a restore in collapse, the chip stays hidden forever until reset.

**Also still broken:** The shrink itself still doesn't look right — the chip content doesn't fade in as the container shrinks. This is because `pillLayer` has no chip content; it's just a white rect. The actual chip is hidden. So there's nothing to "fade in" — it's just white box → invisible.

---

## Root cause summary

There are actually **two separate problems**:

### Problem A: chipEl visibility not restored after collapse
`tpExpandPopover` sets `chipEl.style.visibility = 'hidden'` (line 5574).  
The only restore path is in `tpCollapsePopover`/`tpPopoverCollapseWithVelocity` done blocks.  
Every attempt to remove the chip hiding from collapse also accidentally removed the restore, leaving chips permanently hidden.

**Fix:** The restore `chipEl.style.visibility = ''` must ALWAYS be in the collapse done block. It should restore only `chipEl` (the specific chip that was expanded), not all chips. Keep it there, always.

### Problem B: pillLayer has no chip content — there's nothing to fade in
During collapse the pill layer fades in, but it shows nothing but a white rect with a stroke. The chip's icon + label are not inside it. So the "chip content fading in" effect is impossible with the current DOM.

**The correct approach** (not yet implemented):
- Clone the chip element (or its contents) into `pillLayer` at expand time, positioned to match the chip's location within the panel coordinate space
- OR: keep `chipEl.style.visibility = 'hidden'` for the whole animation but remove the scrim early enough that the chip's transparent background creates no visible seam — the `pillLayer` inset stroke + border-radius match is enough to approximate the chip shape
- OR: accept that `pillLayer` only approximates the chip (stroke + bg match) and focus on making the geometry smooth — the chip "appearing" is just the panel snapping to chip size, panel hides, chip becomes visible at settled state

### Problem C: scrim timing
The independent `springPos(t, SCRIM_STIFFNESS, SCRIM_DAMPING)` spring starts fading immediately from t=0. This means the scrim goes transparent while the panel is still large, exposing the chip at its screen position (with `visibility:hidden` cleared in some attempts) or exposing the gap between panel and chip (in other attempts).

---

## Current state of the code (fixed + still broken)

`tpCollapsePopover` (lines ~5615–5673):
- No chip hiding at start ✓
- Scrim tied to `pXW` progress: `Math.max(0, 1 - Math.max(0, (pXW - 0.6) / 0.4))` ✓
- `chipEl.style.visibility = ''` restore added to done block ✓ **FIXED**

`tpPopoverCollapseWithVelocity` (lines ~6340–6402):
- No chip hiding at start ✓
- Scrim tied to `wProg`: `1 - (simW.x - toW) / Math.max(1, startW - toW)` ✓
- `chipEl.style.visibility = ''` restore added to done block ✓ **FIXED**
- `simScrim` removed ✓

**Still broken:** The shrink animation does not show chip content fading in as the container shrinks. This has never worked across many sessions.

---

## Suggested fix for the next session

### Step 1 — Restore chipEl visibility in both collapse done blocks

In `tpCollapsePopover` done block, add:
```js
if (chipEl) chipEl.style.visibility = '';
```

In `tpPopoverCollapseWithVelocity` done block, also add:
```js
const { chipEl } = tpPopoverState || {};  // NOTE: tpPopoverState is set to null early in this fn
```
Wait — `tpPopoverState` is nulled out at the TOP of `tpPopoverCollapseWithVelocity` (line 6350). The `chipEl` is destructured before the null, so capture it first:
```js
const { fromX: toX, ..., chipEl } = tpPopoverState;  // already done
tpPopoverState = null;  // then nulled
// chipEl is still in scope in tick() closure — restore it in done block:
if (chipEl) chipEl.style.visibility = '';
```

### Step 2 — Decide on pillLayer content strategy

Option A (quick fix): Accept the white rect approximation. The panel snaps to chip size, hides, chip appears. Not smooth but not broken.

Option B (proper fix): At expand time, after `chipEl.style.visibility = 'hidden'`, clone the chip's innerHTML into `pillLayer` and position it to match. During collapse, `pillLayer` fades in showing the actual chip content at the collapsed panel size.

---

## Files
- `/Users/jonas_naimark/Documents/Prototypes/Motion-System/index.html` — single file, all logic
- Expand function: `tpExpandPopover` (~line 5513)
- Collapse function: `tpCollapsePopover` (~line 5615)
- Gesture collapse: `tpPopoverCollapseWithVelocity` (~line 6340)
- Spring helpers: `springPos`, `isSettled`, `SpringSim`, `SCRIM_STIFFNESS/DAMPING` (~line 4430)
- `TP_POPOVER_SPRING = { stiffness: 500, damping: calcDamping(500, 1.0) }` — critically damped
