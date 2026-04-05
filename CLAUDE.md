# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Taiko is a single-page task manager web app hosted on GitHub Pages. The entire application lives in a **single file: `index.html`** (~3,500+ lines). There is no build system, no npm, no TypeScript — just HTML/CSS/JS with React loaded via CDN.

## Running Locally

Open `index.html` directly in a browser. No server or build step required.

Deploy happens automatically via GitHub Actions (`.github/workflows/static.yml`) on every push to `main`.

## Tech Stack

- **React 18** (UMD via CDN, no JSX compilation step — uses Babel Standalone in-browser)
- **Tailwind CSS** (CDN)
- **LZ-String** — URL-safe compression for shortlinks
- **Web Audio API** — procedurally generated taiko drum sounds
- **Canvas API** — animated particle background (Node Garden)
- **LocalStorage** — all persistence; no backend

## Architecture

All code lives inside the `<script type="text/babel">` tag in `index.html`. The structure, top-to-bottom:

1. **Constants & Config** — `APP_VERSION`, `RELEASE_NOTES`, priority/size/status enums, timezone list, audio config
2. **Utilities** — `playTaiko()`, `playTone()`, `parseHHMMSSDuration()`, priority sort helpers
3. **AppProvider** — single React context holding all state: `tasks`, `settings`, `projects`, `filters`. Persists to LocalStorage. Contains export/import/shortlink logic.
4. **Custom Hooks** — `useAppContext`, `usePrevious`, `useNow`, `useWindowWidth`
5. **Helper Components** — `LiveClock`, `TaikoLogo`, `ChevronDownIcon`
6. **TaskCard** — master detail view; handles editing, checklists, dependencies, time tracking, notes
7. **TaskListItem** — compact single row in the task list
8. **TaskList** — groups tasks by project with collapsible sections
9. **FilterToolbar** — priority/project filters + search
10. **HeaderToolbar** — settings, summary, shortlink, dependency graph, delete-all
11. **Modals** — `AddTaskModal`, `NewProjectModal`, `SettingsModal`, `SummaryModal`, `ShortLinkModal`, `DependencyGraphModal`, `ImportModal`, `ReleaseNotesModal`, confirmation dialogs
12. **App** — layout root; master-detail split on desktop, single-pane on mobile
13. **ReactDOM.render** call at the bottom

## Data Model

**Task fields:** `id`, `title`, `notes`, `priority` (`HIGH`/`MEDIUM`/`LOW`), `status` (`To Do`/`Completed`), `project`, `tags`, `size`, `checklist` (items with timestamps), `dependencies` (`{ upstream: [], downstream: [] }`), `dueDate`, `reminders`, time tracking fields.

**AppProvider state:** `tasks[]`, `projectSettings{}`, `appSettings{}`, `collapsedProjects{}`, filter state.

## Key Behaviors to Know

- **Keyboard shortcuts** in the task list: `d` = delete, `c` = complete, `a` = add task
- **Dependency graph** uses Mermaid for rendering
- **Shortlinks** encode full task data as LZ-String compressed JSON in the URL hash
- **Due date urgency** is color-coded: red = overdue, amber = due today, yellow = due soon
- **Circular dependency prevention** is enforced at add-dependency time
- Version bump + `RELEASE_NOTES` entry required for each meaningful release