# Motion System

A single-file interactive prototype documenting Airbnb's motion design system — spring tokens, animation curves, gesture behaviors, and micro-interactions.

## What's in here

**`index.html`** — the whole prototype. Open it in a browser; no build step needed.

### Sections

- **Spring Tokens** — interactive spring card grid showing all speed × character combinations (slow/default/fast/snappy × smooth/small.bounce/medium.bounce/large.bounce). Live demos for sheet, nav transition, grow, bottom bar, modal, and toast.
- **Curves** — easing curves (standard, linear, enter, exit) with live demos.
- **Transitions** — grow popover chip, grow sheet/modal, inset sheet dismiss.
- **Gestures** — scroll momentum, sheet detents, inset sheet dismiss with drag.
- **Micro-interactions** — collapsible animation theme groups with card demos.

## Spring system

Springs are defined by two axes:

| Speed | Stiffness |
|-------|-----------|
| slow | 100 |
| default | 300 |
| fast | 500 |
| snappy | 750 |

| Character | Damping ratio |
|-----------|--------------|
| smooth | 1.0 (critically damped) |
| small.bounce | 0.85 |
| medium.bounce | 0.70 |
| large.bounce | 0.50 |

`damping = 2 × ratio × √stiffness`

## Files

```
index.html          — main prototype
loading-fade.html   — loading fade experiment
templates.html      — component templates
assets/             — images and media
fonts/              — Cereal font (embedded in index.html too)
CLAUDE.md           — AI assistant context
```
