# Taiko: Focus Mode, Staleness Indicators & Theme System

**Date:** 2026-04-21
**Status:** Approved

---

## Overview

Three new features for the Taiko task manager:

1. **Focus Mode** — a split-panel immersive view for working on a single task with a live timer and reticle decoration
2. **Task Staleness Indicators** — age badges on incomplete tasks to surface forgotten work
3. **Theme & Font System** — preset color themes and font choices persisted in settings

All code lives in `index.html` inside the `<script type="text/babel">` tag. No build system.

---

## Feature 1 — Focus Mode

### Trigger

A focus button added to the `HeaderToolbar` component. Icon: a crosshair/target (SVG inline). Clicking it enters focus mode.

**Task selection:** The currently open task in the detail panel is used. If no task is selected, the highest-priority incomplete task is auto-selected (sorted by the existing priority sort order, then by creation date ascending).

### Layout

Focus mode replaces the normal App layout without unmounting it — a boolean `focusMode` state is added to `AppProvider` (or local `App` state). When active:

- The task list column remains in the DOM but dims to `opacity: 0.13` via a CSS class
- A vertical divider renders in the task's priority color (red/amber/blue/slate) between the list and detail columns
- The detail column switches to the focus panel UI

### Focus Panel

Top to bottom:
1. Small `FOCUSING` label — `font-size: 9px`, `font-weight: 700`, `letter-spacing: 2px`, uppercase, in the task's priority color at 60% opacity
2. Task title — `font-size: 10–11px`, muted color
3. **Timer** — `font-size: 38px`, `font-weight: 900`, `letter-spacing: 2px`, full white, `font-variant-numeric: tabular-nums`. Always readable, never obscured.
4. Corner bracket reticle + tick marks (see below)
5. Checklist items if present — interactive, same render as TaskCard checklist

### Reticle & Tick Marks

Four L-shaped corner brackets frame the content area using absolutely-positioned SVG elements. The brackets pulse between `opacity: 0.25` and `opacity: 0.55` on a 3s ease-in-out cycle.

Tick marks accumulate along the top edge between the top-left and top-right brackets:
- One tick mark every 10 seconds (a short vertical line, `stroke: priority-color`, `opacity: 0.65`)
- Maximum 6 ticks (= 1 minute), then the row resets
- A small minute counter label (`Xm`) appears to the left of the tick row once the first minute completes
- Tick state is derived from `elapsedSeconds` in the timer — no separate state needed

### Timer Behavior

- `elapsedSeconds` state initialized to `0` on focus entry; incremented each second via `setInterval`
- If the task already has time tracked (`timeSpent` field), the display shows only the current session time — it does not add to the existing value until exit
- On focus exit: `elapsedSeconds` is added to the task's `timeSpentSeconds` via the existing `updateTask` function
- A soft taiko drum sound plays on enter (via existing `playTaiko()`) and on exit

### Exit

- ESC key listener added while in focus mode
- A small `×` close button in the top-right corner of the focus panel
- Both call the same `exitFocus()` handler which logs time and clears `focusMode` state

---

## Feature 2 — Task Staleness Indicators

### Location

Added to the `TaskListItem` component, rendered alongside the existing due date / priority indicators.

### Logic

Staleness is based on task creation date. Task IDs are `crypto.randomUUID()` strings — not timestamps. A `createdAt: Date.now()` field must be added to the `addTask` function. Existing tasks without `createdAt` show no staleness badge (treated as having unknown age). Only shown on tasks with `status !== 'Completed'`.

| Age | Display |
|---|---|
| < 3 days | Nothing |
| 3–6 days | Small gray pill: `3d`, `opacity: 0.6` |
| 7–13 days | Amber pill: `7d`, amber text + faint amber background |
| 14+ days | Red pill: `14d+`, red text + faint red background + subtle pulse animation |

Pill styles: `font-size: 9px`, `font-weight: 600`, `padding: 1px 5px`, `border-radius: 9999px`.

The `useNow` hook already exists in the codebase — use it to get current time for the age calculation so the indicator updates in real time without extra state.

---

## Feature 3 — Theme & Font System

### Storage

Two new fields added to `appSettings` (already persisted to localStorage):
- `theme: string` — one of `'slate' | 'void' | 'ember' | 'forest' | 'aurora' | 'custom'`
- `font: string` — one of `'inter' | 'jetbrains' | 'space-grotesk' | 'dm-sans' | 'sora'`
- `customTheme: { bg: string, surface: string, accent: string }` — only used when `theme === 'custom'`

### CSS Custom Properties

A `ThemeProvider` component (or effect inside `AppProvider`) injects a `<style>` tag into `document.head` whenever `appSettings.theme` or `appSettings.font` changes. The tag defines CSS custom properties on `:root`:

```css
:root {
  --bg: #0f172a;
  --surface: #1e293b;
  --surface-2: #273346;
  --border: #334155;
  --text-primary: #f1f5f9;
  --text-muted: #94a3b8;
  --accent: #f87171;       /* high priority */
  --font-family: 'Inter', ui-sans-serif, system-ui;
}
```

Existing Tailwind hardcoded color classes are progressively replaced with `var(--bg)` etc. in the inline style props that control layout. Tailwind color utilities on leaf elements (badges, text colors) remain as-is since they don't need theming — only the structural background and surface colors are themed.

### Preset Themes

| Name | `--bg` | `--surface` | `--surface-2` | `--border` |
|---|---|---|---|---|
| Slate | `#0f172a` | `#1e293b` | `#273346` | `#334155` |
| Void | `#050505` | `#111111` | `#1a1a1a` | `#222222` |
| Ember | `#120a06` | `#1c1008` | `#2d1a0a` | `#3d2510` |
| Forest | `#061209` | `#0d1f12` | `#1a3320` | `#1f4a27` |
| Aurora | `#08091a` | `#12133a` | `#1e1f5e` | `#2d2f7a` |

### Fonts

| Name | CSS value | Google Fonts URL required |
|---|---|---|
| Inter | `'Inter', ui-sans-serif` | Already loaded |
| JetBrains Mono | `'JetBrains Mono', monospace` | Yes |
| Space Grotesk | `'Space Grotesk', sans-serif` | Yes |
| DM Sans | `'DM Sans', sans-serif` | Yes |
| Sora | `'Sora', sans-serif` | Yes |

Google Fonts `<link>` tags are injected into `document.head` on demand when a non-Inter font is selected, only if not already present.

### Custom Color Picker

In the Settings modal theme section, a collapsed "Customize" panel sits below the preset swatches. It exposes three `<input type="color">` fields:
- Background
- Surface
- Accent

Changing any field sets `theme: 'custom'` and updates `customTheme`. The custom values flow through the same CSS custom property injection path. `--surface-2` and `--border` are derived automatically from `--surface` by blending toward white at 6% and 12% respectively, so the user only needs 3 inputs.

### Settings Modal UI

A new "Appearance" section added to `SettingsModal`:
1. **Theme** — 5 clickable swatches (colored squares with name labels), selected state shown by a ring. "Customize" toggle below reveals the color pickers.
2. **Font** — 5 buttons showing the font name rendered in that font at ~16px. Selected state shown by a ring.

---

## Data Model Changes

- `appSettings` gains: `theme`, `font`, `customTheme`
- No changes to the `Task` schema — staleness is derived from existing `id` (timestamp) field
- `focusMode` is UI-only state in `App` — not persisted

---

## Version Bump

A new entry in `RELEASE_NOTES` and `APP_VERSION` bump required before shipping.
