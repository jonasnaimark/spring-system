# Spring Token System

16 spring animation tokens — 4 speeds × 4 characters — for Airbnb's design system.

## Tools

| File | Description |
|------|-------------|
| `index.html` | Hub |
| `playground.html` | Interactive demo with phone mockup and live token picker |
| `grid-cards.html` | Editorial cards grouped by character |
| `grid-curves.html` | 4×4 curve grid |
| `grid-strips.html` | Simultaneous speed comparison strips |
| `grid-compare.html` | Side-by-side token comparator |
| `grid-reference.html` | Developer reference table (Web / iOS / Android) |

## Tokens

|  | Slow (100) | Default (300) | Fast (500) | Snappy (750) |
|--|--|--|--|--|
| **Smooth** (ζ 1.0) | slow-smooth | default-smooth | fast-smooth | snappy-smooth |
| **Small Bounce** (ζ 0.85) | slow-small.bounce | default-small.bounce | fast-small.bounce | snappy-small.bounce |
| **Medium Bounce** (ζ 0.70) | slow-medium.bounce | default-medium.bounce | fast-medium.bounce | snappy-medium.bounce |
| **Large Bounce** (ζ 0.50) | slow-large.bounce | default-large.bounce | fast-large.bounce | snappy-large.bounce |

Stiffness controls speed. Damping ratio controls bounciness. Mass = 1. No dependencies — open any HTML file directly in a browser.
