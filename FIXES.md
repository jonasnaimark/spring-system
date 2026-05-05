# Known Fixes

## Collapse flash when interrupted mid-expand

**Symptom:** The first time you close a zoom transition, it briefly flashes a full-screen frame before playing the collapse animation.

**Cause:** `collapseCard` / `tpCollapseCard` hardcodes the collapse start position to the settled end state (`left:0, top:0, fullWidth, fullHeight`). If the expand animation hasn't fully settled yet, the panel is still mid-flight — so the first collapse frame jumps to full-screen, then immediately starts animating back. This produces a one-frame flash.

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

**Note:** This fix alone was not sufficient — see "Zoom jitter: double-fire from phone click handler" below for the complete fix.

---

## Description text mismatch on page load vs. tab click

**Symptom:** The body copy below a demo section shows stale/wrong text on initial page load, but shows correct text after clicking between tabs.

**Cause:** Two sources of truth — the HTML has hardcoded text in the element, and JS updates it via `switchDemo()` / `switchTpDemo()` on tab click. On a clean load with no URL hash, the JS initializer only runs if a hash is present, so the hardcoded HTML text is shown instead of the registry `description`.

**Fix (two parts):**

1. Empty the hardcoded text in the HTML so there's no stale fallback:
```html
<p class="section-body" id="tpBodyCopy"></p>
```

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

Apply this pattern to any section that uses a registry + tab switcher (TP demos, NG demos, spring demos).

---

## Zoom jitter: double-fire from phone click handler

**Symptom:** On open, the panel flashes one frame of the transition, resets, then plays again. On close, same thing intermittently. The `closest()` guard in the phone click handler was not reliable enough.

**Root cause:** A parent `#tpPhone` click listener called `TP_DEMOS[key].play()` on every click inside the phone. The guard (`e.target.closest('.tp-zoom-card')`) failed when the click target was a *child* of the card (e.g. `.sk-carousel-img`) — `closest` still matched, but timing races with the animation start caused the expand to fire, set `tpExpandState`, and then the bubbled event triggered `play()` again which immediately called collapse. This caused the one-frame flash on both open and close.

**Fix (two parts, both required):**

1. Skip `play()` in the phone handler entirely when the active demo is zoom — cards and panel own all zoom interactions:
```js
document.getElementById('tpPhone').addEventListener('click', (e) => {
  const key = document.querySelector('.tp-tab.active')?.dataset.tp;
  if (key === 'zoom') return; // zoom uses card/panel click handlers only
  TP_DEMOS[key]?.play();
});
```

2. Add `stopPropagation` to card and panel handlers so clicks never reach the phone handler at all:
```js
card.addEventListener('click', (e) => {
  e.stopPropagation();
  // ...
});
document.getElementById('tpExpandPanel').addEventListener('click', (e) => {
  e.stopPropagation();
  // ...
});
```

**Pattern to follow for any future zoom-style demo:** the phone-level click handler should always be gated by demo key, and individual interactive elements should always `stopPropagation`.
