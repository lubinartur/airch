# Design System

## Philosophy

The design system exists to enforce calm.

Every token, every component, every spacing decision should make the interface feel more like a thoughtful document and less like a SaaS product.

---

## Typography

**Scale**
- Large editorial headings for primary thoughts
- Medium weight for supporting context
- Small, quiet text for metadata
- Monospace only for financial figures and data

**Principles**
- Hierarchy through size and weight, not color
- Never more than two type sizes on the same visual level
- Line-height generous — text needs room to breathe
- No decorative fonts

**Primary feeling:** reading a well-edited document, not scanning a dashboard.

---

## Color

**Palette**
- Near-white background — not pure white; slightly warm or slightly cool
- One accent color — used sparingly and with intent
- Text in dark grey, not pure black
- Status colors (attention, warning) used only for genuine signals — never decoration

**Principles**
- Color should carry meaning, not create visual interest
- The accent color appears at most once per screen
- Warning colors are earned — not used for minor states
- Dark mode should feel equally calm, not inverted

---

## Layout

**Grid**
- Strong margins — content does not touch the edges
- Maximum content width — long lines of text do not stretch full screen
- Generous vertical spacing between sections

**Density**
- Low. Space between elements is not wasted space.
- When in doubt, add space rather than remove it.

**Alignment**
- Left-aligned text by default
- Centered only for single primary statements — the one dominant idea
- Right-alignment for numerical data in financial contexts

---

## Spacing

Spacing scale built on multiples of a base unit (e.g. 4px or 8px).

Key rule: internal spacing (within a component) should always be smaller than external spacing (between components). The hierarchy of space is part of the hierarchy of information.

---

## Components

**Brief**
The Morning Brief block. One statement. Generous padding. The most important component in AIRCH. Should feel like the first sentence of a thoughtful letter.

**Dialogue**
Conversation thread. Clean. No messenger chrome. No avatars. Text on white. AIRCH's messages visually distinct but not separated by bubbles.

**Space Card**
Entry point to a domain. Shows: name, one-line status from AIRCH, last activity signal. Nothing more on the surface.

**Thread Indicator**
A subtle connection marker — appears when an item belongs to a cross-domain story. Not a tag. A quiet visual link.

**Metric Card**
Used only when a number has been translated into meaning. Never raw data. Always interpretation first, number second.

**Timeline Entry**
Used in Continuity. Minimal. Date, one sentence, domain indicator. Not a feed.

**Mirror Item**
A belief AIRCH holds about the user. Editable inline. Confidence shown subtly. Not a settings row.

**Silence State**
The empty state. Not an illustration. Not onboarding copy. A quiet message — or nothing at all.

---

## Motion

- Transitions: slow and intentional. Nothing snaps.
- Entrance: fade, not slide. AIRCH is calm, not animated.
- No loading spinners if it can be avoided — skeleton states instead.
- No transitions that feel celebratory or gamified.

---

## What the Design System Prevents

- Cards that compete for equal attention
- Widgets without editorial hierarchy
- Raw metrics displayed without meaning
- Visual decoration that does not carry information
- Color used for branding within the UI
- Animation used for delight rather than orientation
