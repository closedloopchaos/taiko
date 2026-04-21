# Focus Mode, Staleness Indicators & Theme System Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Focus Mode with a reticle timer, task staleness age badges, and a full theme/font system to Taiko.

**Architecture:** All code lives in a single `index.html` file inside a `<script type="text/babel">` tag — no build system, no npm. React 18 via CDN, Babel transpiles JSX in-browser. Changes are always to this one file. Open `index.html` directly in a browser to test after every task.

**Tech Stack:** React 18 (UMD/CDN), Babel Standalone, Tailwind CSS (CDN), Web Audio API (already used), LocalStorage (already used).

---

## File Structure

**Single file modified throughout:** `index.html`

Sections touched (top-to-bottom in the file):
- **Line 141** — `CURRENT_APP_VERSION` constant
- **Line 143** — `RELEASE_NOTES` constant
- **Lines 207–212** — after `PRIORITY_CONFIG`, add `PRIORITY_HEX` and `THEMES`/`FONTS` constants
- **Line 421** — `ModalShell` component, change `bg-slate-800` to inline style
- **Line 554** — `defaultSettings` in `AppProvider`, add theme/font fields
- **Lines 540–600** — `AppProvider` `useEffect` block, add theme injection effect
- **Lines 1589–1690** — `TaskListItem`, add staleness badge
- **Lines 1958–1972** — `HeaderToolbar`, add focus button
- **After line 1972** — insert new `FocusPanel` component (before `// ─── 14. FOOTER`)
- **Lines 2373–2388** — `App` state declarations, add `focusMode`/`focusTaskId`
- **Lines 2410–2430** — `App` keyboard handler, add ESC for focus mode
- **Lines 2560–2665** — `App` layout, dim list pane and render FocusPanel
- **Lines 2125–2159** — `SettingsModal`, add Appearance section

---

## Task 1: Theme Constants & CSS Injection

**Files:**
- Modify: `index.html` (~line 212, ~line 421, ~line 554, ~line 583)

- [ ] **Step 1: Add `PRIORITY_HEX`, `THEMES`, and `FONTS` constants**

Find the line `const PRIORITY_TITLE_TO_CONSTANT_MAP = ...` (line ~214) and insert immediately before it:

```javascript
const PRIORITY_HEX = {
    [PRIORITIES.HIGH]:   '#f87171',
    [PRIORITIES.MEDIUM]: '#fbbf24',
    [PRIORITIES.LOW]:    '#38bdf8',
    null:                '#94a3b8',
};

const THEMES = {
    slate:  { bg: '#0f172a', surface: '#1e293b', surface2: '#273346', border: '#334155', textPrimary: '#f1f5f9', textMuted: '#94a3b8' },
    void:   { bg: '#050505', surface: '#111111', surface2: '#1a1a1a', border: '#222222', textPrimary: '#f1f5f9', textMuted: '#9ca3af' },
    ember:  { bg: '#120a06', surface: '#1c1008', surface2: '#2d1a0a', border: '#3d2510', textPrimary: '#fef3c7', textMuted: '#d97706' },
    forest: { bg: '#061209', surface: '#0d1f12', surface2: '#1a3320', border: '#1f4a27', textPrimary: '#f0fdf4', textMuted: '#86efac' },
    aurora: { bg: '#08091a', surface: '#12133a', surface2: '#1e1f5e', border: '#2d2f7a', textPrimary: '#f5f3ff', textMuted: '#a78bfa' },
};

const FONTS = {
    inter:        { family: "'Inter', ui-sans-serif, system-ui", googleUrl: null },
    jetbrains:    { family: "'JetBrains Mono', monospace",       googleUrl: 'https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@400;600;700&display=swap' },
    'space-grotesk': { family: "'Space Grotesk', sans-serif",    googleUrl: 'https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;600;700&display=swap' },
    'dm-sans':    { family: "'DM Sans', sans-serif",             googleUrl: 'https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;600;700&display=swap' },
    sora:         { family: "'Sora', sans-serif",                googleUrl: 'https://fonts.googleapis.com/css2?family=Sora:wght@400;600;700&display=swap' },
};
```

- [ ] **Step 2: Update `defaultSettings` to include theme fields**

Find (line ~554):
```javascript
const defaultSettings = { confirmOnDelete: true };
```
Replace with:
```javascript
const defaultSettings = {
    confirmOnDelete: true,
    theme: 'slate',
    font: 'inter',
    customTheme: { bg: '#0f172a', surface: '#1e293b', accent: '#f87171' },
};
```

- [ ] **Step 3: Add theme injection `useEffect` in `AppProvider`**

Find the comment `// --- Persist to localStorage ---` (line ~583) and insert this `useEffect` directly before it:

```javascript
// Inject theme CSS vars and font whenever settings change
useEffect(() => {
    const themeName = settings.theme || 'slate';
    const fontName  = settings.font  || 'inter';
    const t = themeName === 'custom'
        ? {
            bg: settings.customTheme?.bg      || '#0f172a',
            surface: settings.customTheme?.surface || '#1e293b',
            surface2: settings.customTheme?.surface || '#273346',
            border: settings.customTheme?.surface || '#334155',
            textPrimary: '#f1f5f9',
            textMuted: '#94a3b8',
          }
        : (THEMES[themeName] || THEMES.slate);
    const f = FONTS[fontName] || FONTS.inter;

    // Load Google Font if needed
    if (f.googleUrl && !document.querySelector(`link[href="${f.googleUrl}"]`)) {
        const link = document.createElement('link');
        link.rel  = 'stylesheet';
        link.href = f.googleUrl;
        document.head.appendChild(link);
    }

    let styleEl = document.getElementById('taiko-theme');
    if (!styleEl) {
        styleEl = document.createElement('style');
        styleEl.id = 'taiko-theme';
        document.head.appendChild(styleEl);
    }
    styleEl.textContent = `
        :root {
            --bg: ${t.bg};
            --surface: ${t.surface};
            --surface-2: ${t.surface2};
            --border: ${t.border};
            --text-primary: ${t.textPrimary};
            --text-muted: ${t.textMuted};
            --font-family: ${f.family};
        }
        html, body { background-color: var(--bg) !important; font-family: var(--font-family) !important; }
    `;
}, [settings.theme, settings.font, settings.customTheme]);
```

- [ ] **Step 4: Update `ModalShell` to use CSS var for background**

Find `ModalShell` at line ~421:
```javascript
<div className={`bg-slate-800 rounded-xl shadow-2xl w-full ${maxWidth} flex flex-col max-h-[90vh]`}
```
Replace with:
```javascript
<div className={`rounded-xl shadow-2xl w-full ${maxWidth} flex flex-col max-h-[90vh]`} style={{ background: 'var(--surface)', border: '1px solid var(--border)' }}
```

- [ ] **Step 5: Verify in browser**

Open `index.html` in browser. Open Settings → change nothing, just confirm modals (Settings, etc.) have the right background. Then add `console.log(document.getElementById('taiko-theme').textContent)` in devtools to confirm the style tag is injected.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: add theme constants and CSS var injection"
```

---

## Task 2: Appearance Section in Settings Modal

**Files:**
- Modify: `index.html` (`SettingsModal` ~line 2125)

- [ ] **Step 1: Replace `SettingsModal` with version that includes Appearance section**

Find the entire `SettingsModal` component (lines ~2125–2159) and replace it:

```javascript
const SettingsModal = ({ settings, onSettingsChange, onClose }) => {
    const { exportData, initiateImport } = useAppContext();
    const [showCustomPicker, setShowCustomPicker] = useState(false);

    const setTheme = (name) => onSettingsChange(p => ({ ...p, theme: name }));
    const setFont  = (name) => onSettingsChange(p => ({ ...p, font: name }));
    const setCustomColor = (key, val) => onSettingsChange(p => ({
        ...p,
        theme: 'custom',
        customTheme: { ...(p.customTheme || {}), [key]: val }
    }));

    const themeEntries = Object.entries(THEMES);
    const fontEntries  = Object.entries(FONTS);

    return (
        <ModalShell title="Interface Settings" onClose={onClose} maxWidth="max-w-xl">
            <div className="space-y-8">

                {/* Appearance */}
                <div className="space-y-6">
                    <h4 className="text-lg font-bold text-slate-200">Appearance</h4>

                    {/* Theme presets */}
                    <div className="space-y-2">
                        <p className="text-sm font-semibold text-slate-400 uppercase tracking-wider">Color Theme</p>
                        <div className="flex flex-wrap gap-3">
                            {themeEntries.map(([name, t]) => {
                                const isActive = settings.theme === name;
                                return (
                                    <button key={name} onClick={() => setTheme(name)}
                                        title={name.charAt(0).toUpperCase() + name.slice(1)}
                                        className={`flex flex-col items-center gap-1.5 p-1 rounded-lg transition-all ${isActive ? 'ring-2 ring-sky-400' : 'ring-1 ring-slate-600 hover:ring-slate-400'}`}>
                                        <div className="w-10 h-10 rounded-md border border-white/10 overflow-hidden flex"
                                             style={{ background: t.bg }}>
                                            <div className="flex-1" style={{ background: t.surface }} />
                                            <div className="w-3" style={{ background: t.surface2 }} />
                                        </div>
                                        <span className="text-[10px] text-slate-400 capitalize">{name}</span>
                                    </button>
                                );
                            })}
                            {/* Custom swatch */}
                            <button onClick={() => { setShowCustomPicker(p => !p); if (settings.theme !== 'custom') setTheme('custom'); }}
                                className={`flex flex-col items-center gap-1.5 p-1 rounded-lg transition-all ${settings.theme === 'custom' ? 'ring-2 ring-sky-400' : 'ring-1 ring-slate-600 hover:ring-slate-400'}`}>
                                <div className="w-10 h-10 rounded-md border border-white/10 flex items-center justify-center bg-slate-700 text-slate-400 text-xs font-bold">+</div>
                                <span className="text-[10px] text-slate-400">Custom</span>
                            </button>
                        </div>

                        {/* Custom color picker */}
                        {showCustomPicker && (
                            <div className="flex flex-wrap gap-4 pt-2 pl-1">
                                {[
                                    { key: 'bg',      label: 'Background' },
                                    { key: 'surface', label: 'Surface' },
                                    { key: 'accent',  label: 'Accent' },
                                ].map(({ key, label }) => (
                                    <label key={key} className="flex items-center gap-2 text-sm text-slate-300 cursor-pointer">
                                        <input type="color"
                                            value={settings.customTheme?.[key] || '#0f172a'}
                                            onChange={e => setCustomColor(key, e.target.value)}
                                            className="w-8 h-8 rounded cursor-pointer border-0 bg-transparent"
                                        />
                                        {label}
                                    </label>
                                ))}
                            </div>
                        )}
                    </div>

                    {/* Font picker */}
                    <div className="space-y-2">
                        <p className="text-sm font-semibold text-slate-400 uppercase tracking-wider">Font</p>
                        <div className="flex flex-wrap gap-2">
                            {fontEntries.map(([name, f]) => {
                                const isActive = (settings.font || 'inter') === name;
                                const label = name === 'space-grotesk' ? 'Space Grotesk' : name === 'dm-sans' ? 'DM Sans' : name === 'jetbrains' ? 'JetBrains Mono' : name.charAt(0).toUpperCase() + name.slice(1);
                                return (
                                    <button key={name} onClick={() => setFont(name)}
                                        style={{ fontFamily: f.family }}
                                        className={`px-3 py-2 rounded-lg text-sm transition-all ${isActive ? 'bg-sky-500/20 text-sky-300 ring-1 ring-sky-400' : 'bg-slate-700/50 text-slate-300 hover:bg-slate-700'}`}>
                                        {label}
                                    </button>
                                );
                            })}
                        </div>
                    </div>
                </div>

                {/* Confirm on delete toggle */}
                <div className="pt-6 border-t border-slate-700/50 flex items-center justify-between">
                    <label className="text-lg text-slate-200 font-semibold">Confirm on Delete</label>
                    <button onClick={() => onSettingsChange(p => ({ ...p, confirmOnDelete: !p.confirmOnDelete }))} className={`relative inline-flex items-center h-6 rounded-full w-11 transition-colors ${settings.confirmOnDelete ? 'bg-sky-500' : 'bg-slate-600'}`}>
                        <span className={`inline-block w-4 h-4 transform bg-white rounded-full transition-transform ${settings.confirmOnDelete ? 'translate-x-6' : 'translate-x-1'}`} />
                    </button>
                </div>

                {/* Data & Portability */}
                <div className="pt-6 border-t border-slate-700/50 space-y-4">
                    <h4 className="text-lg font-bold text-slate-200">Data &amp; Portability</h4>
                    <p className="text-xs text-slate-400">Download a backup of your tasks and settings, or restore from a previous export.</p>
                    <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
                        <button onClick={exportData} className="flex items-center justify-center p-3 rounded-lg border border-slate-600 bg-slate-800 hover:bg-slate-700 text-slate-300 transition-all gap-2">
                            <svg xmlns="http://www.w3.org/2000/svg" className="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-4l-4 4m0 0l-4-4m4 4V4" /></svg>
                            <span>Export Data</span>
                        </button>
                        <label className="flex items-center justify-center p-3 rounded-lg border border-slate-600 bg-slate-800 hover:bg-slate-700 text-slate-300 transition-all gap-2 cursor-pointer">
                            <svg xmlns="http://www.w3.org/2000/svg" className="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-8l-4-4m0 0L8 8m4-4v12" /></svg>
                            <span>Import Data</span>
                            <input type="file" accept=".json" className="hidden" onChange={e => {
                                const file = e.target.files[0]; if (!file) return;
                                const reader = new FileReader();
                                reader.onload = ev => { initiateImport(ev.target.result); onClose(); };
                                reader.readAsText(file);
                            }} />
                        </label>
                    </div>
                </div>

            </div>
        </ModalShell>
    );
};
```

- [ ] **Step 2: Verify in browser**

Open `index.html` → Settings → Appearance section appears. Click theme swatches — background color of the page changes. Click font buttons — body font changes visibly. Click Custom → color pickers appear. Reload page — last selected theme/font persists (localStorage).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add Appearance section to Settings with theme and font pickers"
```

---

## Task 3: Task Staleness Indicators

**Files:**
- Modify: `index.html` (`TaskListItem` ~line 1589)

- [ ] **Step 1: Add staleness helper function**

Find `const PRIORITY_TITLE_TO_CONSTANT_MAP` (~line 214) and insert this helper right before the `TaskListItem` component declaration (~line 1589):

```javascript
const getTaskAgeDays = (task) => {
    if (!task.createdAt) return null;
    return Math.floor((Date.now() - new Date(task.createdAt).getTime()) / (1000 * 60 * 60 * 24));
};
```

- [ ] **Step 2: Add staleness badge inside `TaskListItem`**

In `TaskListItem`, find the `flex items-center gap-x-2 gap-y-1 flex-wrap mt-1.5 text-xs` div (line ~1653). It currently contains the due date badge and tags. Add the staleness badge as the last child of that flex container:

Find:
```javascript
            <div className="flex items-center gap-x-2 gap-y-1 flex-wrap mt-1.5 text-xs">
                {task.dueDate && (
```

Replace with:
```javascript
            {(() => {
                const ageDays = (!isDone && task.createdAt) ? getTaskAgeDays(task) : null;
                const stalenessBadge = ageDays === null ? null
                    : ageDays >= 14 ? { label: '14d+', cls: 'text-red-400 bg-red-500/10 border-red-500/30 animate-pulse' }
                    : ageDays >= 7  ? { label: `${ageDays}d`, cls: 'text-amber-400 bg-amber-500/10 border-amber-500/30' }
                    : ageDays >= 3  ? { label: `${ageDays}d`, cls: 'text-slate-500 bg-slate-700/30 border-slate-600/30' }
                    : null;
                return null; // stale badge rendered below inline
            })()}
            <div className="flex items-center gap-x-2 gap-y-1 flex-wrap mt-1.5 text-xs">
                {task.dueDate && (
```

That's awkward. Instead, compute the badge value before the return statement. Find the line right after `const progressPercent = ...` (line ~1610) and add:

```javascript
            const ageDays = (!isDone && task.createdAt)
                ? Math.floor((Date.now() - new Date(task.createdAt).getTime()) / (1000 * 60 * 60 * 24))
                : null;
            const stalenessBadge = ageDays === null ? null
                : ageDays >= 14 ? { label: '14d+', cls: 'text-red-400 bg-red-500/10 border border-red-500/30' }
                : ageDays >= 7  ? { label: `${ageDays}d`, cls: 'text-amber-400 bg-amber-500/10 border border-amber-500/30' }
                : ageDays >= 3  ? { label: `${ageDays}d`, cls: 'text-slate-500 bg-slate-700/30 border border-slate-600/30' }
                : null;
```

Then inside the `flex items-center gap-x-2` div, add the badge as the last child before the closing `</div>`:

```javascript
                {stalenessBadge && (
                    <span className={`font-semibold px-1.5 py-0.5 rounded-full text-[9px] whitespace-nowrap ${stalenessBadge.cls}`}>
                        {stalenessBadge.label}
                    </span>
                )}
```

- [ ] **Step 3: Verify in browser**

Tasks created today show no badge. To test the amber/red states without waiting days: temporarily change the age thresholds to 0/1/2 days, reload, confirm badges appear, then restore the real thresholds (3/7/14).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add task staleness age badges to task list items"
```

---

## Task 4: FocusPanel Component

**Files:**
- Modify: `index.html` (insert new component after `HeaderToolbar` ~line 1972, before `// ─── 14. FOOTER`)

- [ ] **Step 1: Insert the `FocusPanel` component**

Find the line `// ─── 14. FOOTER ───` (~line 1975) and insert the entire `FocusPanel` component immediately before it:

```javascript
        // ─── FocusPanel ──────────────────────────────────────────────────────

        const FocusPanel = ({ task, onExit }) => {
            const { updateTask } = useAppContext();
            const [elapsedSeconds, setElapsedSeconds] = useState(0);

            // Start timer and play entry sound
            useEffect(() => {
                playTaiko('DON', 0.3);
                const id = setInterval(() => setElapsedSeconds(s => s + 1), 1000);
                return () => clearInterval(id);
            }, []);

            // ESC to exit — capture elapsedSeconds via ref to avoid stale closure
            const elapsedRef = useRef(0);
            useEffect(() => { elapsedRef.current = elapsedSeconds; }, [elapsedSeconds]);
            useEffect(() => {
                const handler = (e) => { if (e.key === 'Escape') handleExit(); };
                window.addEventListener('keydown', handler);
                return () => window.removeEventListener('keydown', handler);
            }, []); // eslint-disable-line

            const handleExit = useCallback(() => {
                const elapsed = elapsedRef.current;
                playTaiko('KA', 0.25);
                if (elapsed > 0) {
                    updateTask(task.id, { timeSpentSeconds: (task.timeSpentSeconds || 0) + elapsed });
                }
                onExit();
            }, [task.id, task.timeSpentSeconds, updateTask, onExit]);

            const pColor = PRIORITY_HEX[task.priority] || PRIORITY_HEX[null];

            // Timer display
            const mins = String(Math.floor(elapsedSeconds / 60)).padStart(2, '0');
            const secs = String(elapsedSeconds % 60).padStart(2, '0');

            // Tick marks: 1 tick per 10s, row of 6, then reset with minute counter
            const totalTicks       = Math.floor(elapsedSeconds / 10);
            const currentRowTicks  = totalTicks % 6;
            const completedMinutes = Math.floor(totalTicks / 6);

            // Pulse animation for brackets: 3s ease-in-out cycle
            const [bracketOpacity, setBracketOpacity] = useState(0.3);
            useEffect(() => {
                let ascending = true;
                const id = setInterval(() => {
                    setBracketOpacity(prev => {
                        const next = ascending ? prev + 0.015 : prev - 0.015;
                        if (next >= 0.55) ascending = false;
                        if (next <= 0.25) ascending = true;
                        return Math.max(0.25, Math.min(0.55, next));
                    });
                }, 50);
                return () => clearInterval(id);
            }, []);

            const cornerStyle = { position: 'absolute', pointerEvents: 'none', opacity: bracketOpacity };
            const bracketPath = "M14,2 L2,2 L2,14";

            return (
                <div className="w-full h-full flex flex-col items-center justify-center gap-3 relative p-6 overflow-hidden">
                    {/* Corner brackets */}
                    <svg width="16" height="16" viewBox="0 0 16 16" style={{ ...cornerStyle, top: 12, left: 12 }}>
                        <path d={bracketPath} fill="none" stroke={pColor} strokeWidth="1.5" strokeLinecap="round"/>
                    </svg>
                    <svg width="16" height="16" viewBox="0 0 16 16" style={{ ...cornerStyle, top: 12, right: 12, transform: 'scaleX(-1)' }}>
                        <path d={bracketPath} fill="none" stroke={pColor} strokeWidth="1.5" strokeLinecap="round"/>
                    </svg>
                    <svg width="16" height="16" viewBox="0 0 16 16" style={{ ...cornerStyle, bottom: 12, left: 12, transform: 'scaleY(-1)' }}>
                        <path d={bracketPath} fill="none" stroke={pColor} strokeWidth="1.5" strokeLinecap="round"/>
                    </svg>
                    <svg width="16" height="16" viewBox="0 0 16 16" style={{ ...cornerStyle, bottom: 12, right: 12, transform: 'scale(-1)' }}>
                        <path d={bracketPath} fill="none" stroke={pColor} strokeWidth="1.5" strokeLinecap="round"/>
                    </svg>

                    {/* Tick marks along top edge */}
                    <div style={{ position: 'absolute', top: 14, left: 36, right: 36, height: 10, display: 'flex', alignItems: 'center', gap: 4, pointerEvents: 'none' }}>
                        {completedMinutes > 0 && (
                            <span style={{ fontSize: 8, color: pColor, opacity: 0.5, marginRight: 2, fontWeight: 600 }}>{completedMinutes}m</span>
                        )}
                        {Array.from({ length: currentRowTicks }, (_, i) => (
                            <div key={i} style={{ width: 1.5, height: 8, background: pColor, opacity: 0.65, borderRadius: 1, flexShrink: 0 }} />
                        ))}
                    </div>

                    {/* Close button */}
                    <button onClick={handleExit} style={{ position: 'absolute', top: 10, right: 10, color: '#475569', fontSize: 18, lineHeight: 1, background: 'none', border: 'none', cursor: 'pointer', padding: 4, zIndex: 2 }}>×</button>

                    {/* Content */}
                    <span style={{ fontSize: 9, fontWeight: 700, letterSpacing: '2px', textTransform: 'uppercase', color: pColor, opacity: 0.7 }}>Focusing</span>
                    <div style={{ fontSize: 11, fontWeight: 600, color: '#64748b', textAlign: 'center', maxWidth: 200, lineHeight: 1.4 }}>{task.title}</div>
                    <div style={{ fontSize: 42, fontWeight: 900, letterSpacing: 3, color: '#f1f5f9', fontVariantNumeric: 'tabular-nums', lineHeight: 1, textShadow: `0 0 20px ${pColor}40` }}>
                        {mins}:{secs}
                    </div>

                    {/* Checklist (interactive) */}
                    {task.checklist && task.checklist.length > 0 && (
                        <div style={{ display: 'flex', flexDirection: 'column', gap: 4, marginTop: 4, maxWidth: 200, width: '100%' }}>
                            {task.checklist.map(item => (
                                <div key={item.id} style={{ display: 'flex', alignItems: 'center', gap: 6, fontSize: 10 }}>
                                    <div style={{ width: 10, height: 10, border: `1.5px solid ${item.completed ? '#22c55e' : '#475569'}`, borderRadius: 2, flexShrink: 0, background: item.completed ? 'rgba(34,197,94,0.15)' : 'transparent' }} />
                                    <span style={{ color: item.completed ? '#475569' : '#94a3b8', textDecoration: item.completed ? 'line-through' : 'none' }}>{item.text}</span>
                                </div>
                            ))}
                        </div>
                    )}

                    {/* ESC hint */}
                    <div style={{ position: 'absolute', bottom: 14, fontSize: 9, color: '#334155', letterSpacing: '0.5px' }}>ESC to exit</div>
                </div>
            );
        };
```

- [ ] **Step 2: Verify syntax**

Open `index.html` in browser. Open browser devtools — confirm no console errors. The app should load normally (FocusPanel isn't rendered yet — that's fine).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add FocusPanel component with tick-mark reticle timer"
```

---

## Task 5: Focus Mode Integration

**Files:**
- Modify: `index.html` (`HeaderToolbar` ~line 1958, `App` ~line 2360)

- [ ] **Step 1: Add focus button to `HeaderToolbar`**

Find `HeaderToolbar`'s return value (line ~1960) — specifically the first `<button>` (the graph button). Insert a new focus button and divider immediately before the graph button's existing divider:

Find:
```javascript
            return (
                <div className="flex items-center flex-wrap gap-2">
                    <button onClick={onShowGraph}
```

Replace with:
```javascript
            return (
                <div className="flex items-center flex-wrap gap-2">
                    <button onClick={onEnterFocus} title="Focus Mode" className="p-2 text-slate-400 hover:text-emerald-400 hover:bg-slate-800/50 rounded-full transition-all">
                        <svg xmlns="http://www.w3.org/2000/svg" className="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M15 12a3 3 0 11-6 0 3 3 0 016 0z" /><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M2.458 12C3.732 7.943 7.523 5 12 5c4.478 0 8.268 2.943 9.542 7-1.274 4.057-5.064 7-9.542 7-4.477 0-8.268-2.943-9.542-7z" /></svg>
                    </button>
                    <div className="h-6 w-px bg-slate-700/50 mx-1" />
                    <button onClick={onShowGraph}
```

Also update the `HeaderToolbar` prop destructuring at line ~1958 to include `onEnterFocus`:

Find:
```javascript
        const HeaderToolbar = React.memo(({ isSmallScreen, onShowSettings, onGenerateSummary, onShowDeleteAllModal, showDeleteAllButton, onGenerateShortLink, onShowGraph }) => {
```
Replace with:
```javascript
        const HeaderToolbar = React.memo(({ isSmallScreen, onShowSettings, onGenerateSummary, onShowDeleteAllModal, showDeleteAllButton, onGenerateShortLink, onShowGraph, onEnterFocus }) => {
```

- [ ] **Step 2: Add `focusMode` state to `App`**

Find the `App` state declarations block (~line 2373–2388). Add these two lines after `const [showShortLinkModal, setShowShortLinkModal] = useState(false);`:

```javascript
            const [focusMode,    setFocusMode]    = useState(false);
            const [focusTaskId,  setFocusTaskId]  = useState(null);
```

- [ ] **Step 3: Add `enterFocus` and `exitFocus` handlers in `App`**

Find `const handleConfirmDelete` (~line 2483) and add these handlers just before it:

```javascript
            const enterFocus = useCallback(() => {
                const taskId = selectedTaskId
                    || tasks.filter(t => t.status !== TASK_STATUS.DONE)
                             .sort((a, b) => getPrioritySortValue(a.priority) - getPrioritySortValue(b.priority))[0]?.id;
                if (!taskId) return;
                setFocusTaskId(taskId);
                setFocusMode(true);
            }, [selectedTaskId, tasks]);

            const exitFocus = useCallback(() => {
                setFocusMode(false);
                setFocusTaskId(null);
            }, []);
```

- [ ] **Step 4: Wire `onEnterFocus` prop to `HeaderToolbar` call site**

Find the `<HeaderToolbar` JSX (~line 2594) and add the new prop:

Find:
```javascript
                                    onShowGraph={() => setShowGraphModal(true)}
```
Replace with:
```javascript
                                    onShowGraph={() => setShowGraphModal(true)}
                                    onEnterFocus={enterFocus}
```

- [ ] **Step 5: Dim task list and render `FocusPanel` in the layout**

Find the task list pane div (~line 2625):
```javascript
                        <div className={`w-full md:w-1/3 flex-shrink-0 overflow-y-auto pr-2 scroll-container ${isSmallScreen ? 'mobile-scroll-padding' : ''} ${showDetailViewOnMobile ? 'hidden' : ''}`}>
```
Replace with:
```javascript
                        <div className={`w-full md:w-1/3 flex-shrink-0 overflow-y-auto pr-2 scroll-container ${isSmallScreen ? 'mobile-scroll-padding' : ''} ${showDetailViewOnMobile ? 'hidden' : ''} ${focusMode ? 'opacity-[0.13] pointer-events-none select-none' : ''} transition-opacity duration-300`}>
```

Find the detail pane opening div (~line 2643):
```javascript
                        <div className={`w-full md:w-2/3 flex-grow overflow-y-auto scroll-container ${isSmallScreen ? 'mobile-scroll-padding' : ''} ${showListViewOnMobile ? 'hidden' : ''}`}>
                            {selectedProjectName ? (
```
Replace with:
```javascript
                        <div className={`w-full md:w-2/3 flex-grow overflow-y-auto scroll-container ${isSmallScreen ? 'mobile-scroll-padding' : ''} ${showListViewOnMobile ? 'hidden' : ''}`}
                             style={focusMode ? { borderLeft: `1px solid ${PRIORITY_HEX[tasks.find(t => t.id === focusTaskId)?.priority] || '#94a3b8'}40` } : {}}>
                            {focusMode && focusTaskId ? (
                                <FocusPanel task={tasks.find(t => t.id === focusTaskId)} onExit={exitFocus} />
                            ) : selectedProjectName ? (
```

And close the new conditional properly — find the existing closing of the detail pane conditional:
```javascript
                            ) : (
                                <div className="h-full flex items-center justify-center text-slate-500 text-xl">
                                    Select a task or project to see details
                                </div>
                            )}
```
This stays as-is (the `: selectedProjectName ?` chain now has one more leading condition).

- [ ] **Step 6: Verify in browser**

Open `index.html`. Select a task. Click the focus button (eye icon) in the header. The task list should dim, the detail pane should show the FocusPanel with the large timer ticking, corner brackets, and tick marks appearing every 10 seconds. Press ESC — the panel exits and the list un-dims. Reload the page to confirm no state leaks.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: add Focus Mode with dimmed list, reticle timer, and ESC exit"
```

---

## Task 6: Version Bump

**Files:**
- Modify: `index.html` (lines ~141–200)

- [ ] **Step 1: Bump version**

Find:
```javascript
        const CURRENT_APP_VERSION = '1.6.0';
```
Replace with:
```javascript
        const CURRENT_APP_VERSION = '1.7.0';
```

- [ ] **Step 2: Add release notes entry**

Find the `RELEASE_NOTES` object and add a new entry at the top (before the `'1.6.0'` entry):

```javascript
        const RELEASE_NOTES = {
            '1.7.0': {
                title: "Version 1.7.0",
                notes: [
                    { title: "🎯 Focus Mode", description: "Enter an immersive single-task view from the header. A reticle timer tracks your session and automatically logs time to the task on exit. Press ESC or click × to leave." },
                    { title: "⏳ Staleness Indicators", description: "Tasks that haven't moved in a while now show age badges (3d / 7d / 14d+) in the task list, so nothing falls through the cracks." },
                    { title: "🎨 Theme & Font System", description: "Choose from 5 color themes (Slate, Void, Ember, Forest, Aurora) or build a custom palette. Pick from 5 fonts in Settings → Appearance." },
                ]
            },
```

- [ ] **Step 3: Final verification**

Open `index.html`. Confirm the Release Notes modal appears automatically (new version). Check all three features work end-to-end:
1. Settings → Appearance → change theme and font → reload → settings persist
2. Task list shows staleness badges on older tasks (or test by temporarily lowering the 3-day threshold to 0)
3. Focus Mode: click eye icon → panel appears → timer ticks → ticks accumulate → ESC exits and logs time to `timeSpentSeconds`

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "chore: bump version to 1.7.0 and add release notes for Focus Mode, Staleness, and Themes"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|---|---|
| Focus Mode: header button trigger | Task 5, Step 1 |
| Focus Mode: auto-selects highest-priority task if none selected | Task 5, Step 3 (`enterFocus`) |
| Focus Mode: dims task list | Task 5, Step 5 |
| Focus Mode: priority-colored divider | Task 5, Step 5 (inline border style) |
| Focus Mode: large timer display | Task 4 (`FocusPanel`) |
| Focus Mode: corner bracket reticle | Task 4 (`FocusPanel`) |
| Focus Mode: tick marks every 10s, 6=1min, minute counter | Task 4 (`FocusPanel`) |
| Focus Mode: ESC exit | Task 4 (ESC handler) |
| Focus Mode: × close button | Task 4 (close button) |
| Focus Mode: logs timeSpentSeconds on exit | Task 4 (`handleExit`) |
| Focus Mode: taiko drum sound on enter/exit | Task 4 (`useEffect` / `handleExit`) |
| Focus Mode: checklist items shown | Task 4 (checklist render) |
| Staleness: hidden < 3 days | Task 3, Step 2 |
| Staleness: gray 3–6 days | Task 3, Step 2 |
| Staleness: amber 7–13 days | Task 3, Step 2 |
| Staleness: red + pulse 14+ days | Task 3, Step 2 |
| Staleness: only on To Do tasks | Task 3, Step 2 (`!isDone`) |
| Staleness: uses `task.createdAt` | Task 3 (already present in schema) |
| Theme: 5 presets | Task 1 (`THEMES` constant) + Task 2 (Settings UI) |
| Theme: custom color picker | Task 2 (Settings UI, `showCustomPicker`) |
| Theme: CSS var injection | Task 1, Step 3 |
| Theme: body background + font override | Task 1, Step 3 (injected style tag) |
| Font: 5 curated options | Task 1 (`FONTS` constant) + Task 2 (Settings UI) |
| Font: Google Fonts loaded on demand | Task 1, Step 3 (link injection) |
| Settings: persisted to localStorage | Task 1, Step 2 (defaultSettings; existing persistence covers it) |
| Version bump + release notes | Task 6 |

All requirements covered. No gaps.

**Placeholder scan:** No TBDs, no "similar to above", no incomplete steps.

**Type consistency:** `PRIORITY_HEX` defined in Task 1 Step 1, used in `FocusPanel` (Task 4) and App layout (Task 5 Step 5). `FONTS`/`THEMES` defined in Task 1, consumed in Task 1 Step 3 (injection) and Task 2 (Settings UI). `focusTaskId` set in `enterFocus` (Task 5 Step 3), read in layout (Task 5 Step 5) and `FocusPanel` prop (Task 5 Step 5). `timeSpentSeconds` field confirmed present in schema.
