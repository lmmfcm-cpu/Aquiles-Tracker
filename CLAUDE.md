# CLAUDE.md

Guidance for AI assistants working in this repository.

## What this is

**AQUILES TRACKER v2** — a single-user, mobile-first web app for logging weightlifting
sets (exercise, load, reps, RPE), tracking personal records, and running a smart
rest timer between sets. React 18 SPA, no backend: all data lives in
`localStorage`, with optional Google Calendar sync.

The entire product is one ~900-line file. The UI and all identifiers are in
**Spanish** (`es-CL` locale for dates).

## Repository layout

```
Aquiles-Tracker      # THE APP — the whole React component tree (~912 lines, no file extension)
index.html           # HTML shell: Tailwind via CDN, <script type="module" src="/src/index.js">
src/index.js         # React 18 entry point — mounts <App /> from './App'
package.json         # CRA (react-scripts 5) deps + scripts
```

That's the complete file list. There is no `node_modules`, no lockfile, no
`.gitignore`, no README, no tests, and no CI.

## Critical: the app does not currently build or run

Three things are out of sync. Know this before promising anything runs:

1. **The main component file has no extension and the wrong name.**
   `src/index.js` does `import App from './App'`, but there is no `src/App.js`.
   The component lives at repo root in a file literally named `Aquiles-Tracker`.
   No bundler resolves that import.

2. **`package.json` says Create React App; `index.html` says Vite.**
   `react-scripts` expects `public/index.html` and injects its own script tag.
   The root `index.html` instead hand-writes `<script src="/src/index.js" type="module">`,
   which is the Vite convention. Only one of these can be right.

3. **Tailwind comes from the CDN** (`https://cdn.tailwindcss.com` in `index.html`),
   not from a build step. There is no `tailwind.config.js` and no PostCSS setup.
   Every class in the app is a stock Tailwind utility, so the CDN covers it —
   but the CDN build is dev-only and warns in the console.

### Minimal fix (choose one lane, don't mix)

**CRA lane** — matches `package.json` as committed:
```
mkdir public && mv index.html public/index.html
# strip the hand-written <script src="/src/index.js"> line; CRA injects its own
git mv Aquiles-Tracker src/App.js
npm install && npm start
```

**Vite lane** — matches `index.html` as committed:
```
git mv Aquiles-Tracker src/App.jsx
npm i -D vite @vitejs/plugin-react
# replace the react-scripts scripts with vite / vite build / vite preview
```

If a task doesn't require running the app, prefer editing `Aquiles-Tracker`
in place over restructuring the repo. Renaming it is a real decision the
owner should make — raise it, don't do it silently as a side effect.

## Component map (all inside `Aquiles-Tracker`)

| Symbol | Lines | Role |
|---|---|---|
| `calcularDescansoSugerido()` | 13 | Pure fn: rest seconds from exercise + RPE + goal |
| `EntrenadorAquiles()` | 33 | Default export. Root: state, persistence, Google sync, routing, chrome |
| `PantallaEntrada` | 325 | Set-entry form; saves then auto-opens the timer |
| `PantallaHistorial` | 543 | Filterable log list with per-entry delete |
| `PantallaDashboard` | 608 | Stats tiles, CSS bar chart, PRs, recent sessions |
| `TimerModal` | 706 | Countdown, pause, presets, WebAudio beep |
| `TarjetaStats`, `NavButton` | 889, 900 | Small presentational helpers |

Navigation is a `pantalla` string in root state (`'entrada' | 'historial' |
'dashboard' | 'config'`) rendered by conditional blocks — there is no router.
The `'config'` screen is inlined in the root component rather than extracted.

## Data model

One flat array, `entrenamientos`, persisted to `localStorage` under the key
`'entrenamientos'`. Each entry is **one set**, not one session:

```js
{
  id: Date.now(),        // also the React key
  fecha: <epoch ms>,     // form holds 'YYYY-MM-DD'; converted on save
  ejercicio: string,     // must match a name in the `ejercicios` list
  carga: number,         // kg, parseFloat
  reps: number,          // parseInt
  rpe: string,           // '1'–'10' in 0.5 steps — STORED AS A STRING
  variacion: string,     // '' or one of the `variaciones` presets
  esDropSet: boolean,
  notas: string,
  ronda: number,         // 1–5
  objetivo: 'fuerza' | 'hipertrofia' | 'resistencia'
}
```

Volume is always derived, never stored: `carga * reps`.
The `ejercicios` catalog is a hardcoded 24-item array in root state — adding an
exercise means editing that literal.

## The rest-timer rule (the app's one piece of domain logic)

`calcularDescansoSugerido()` classifies the exercise into heavy compound (180s),
polyarticular (150s), isolation (60s), or default (90s), then adjusts:
`RPE ≥ 9.5` +60s, `RPE ≥ 9` +30s (these stack), `RPE ≤ 6` −30s; goal `fuerza`
+30s, `hipertrofia` −15s. Floored at 45s.

Exercise classification is by **exact string match** against three arrays. A new
exercise name that isn't in any of them silently falls through to the 90s
default — if you add exercises, add them to the right bucket too.

## Google integration

OAuth 2.0 implicit flow, hand-rolled, no library. The token is read from and
written to `localStorage` under `'google_token'`; sets sync to Calendar via a
raw `fetch` POST to `calendar/v3/calendars/primary/events`.

`GOOGLE_CLIENT_ID` at line 5 is the placeholder `'YOUR_CLIENT_ID.apps.googleusercontent.com'`.
Sync is dead until it's replaced. **Do not commit a real client ID** — move it to
an env var if you wire this up. The `/callback` redirect route has no handler
anywhere, so the token is never actually captured from the URL fragment; that
half of the flow is unimplemented. Drive and Sheets scopes are requested but
unused.

## Conventions

- **Spanish identifiers throughout.** `guardarEntrenamiento`, `pantalla`,
  `carga`, `tiempoRestante`. Match this — do not introduce English names.
- **Emoji in UI copy** is the house style (`🏋️`, `💪`, `📅`). Keep it.
- **Tailwind only**, inline in `className`. No CSS files, no styled-components.
  Palette: `slate-700/600` surfaces on a `slate-900`→black gradient, `orange-*`
  for emphasis/active, `red-600`→`orange-600` header, `green-*` for confirm.
- **`useState` only.** No reducers, no context, no external state library.
- Persistence is manual: every mutation writes `localStorage.setItem` at the
  call site. There is no save-on-change effect — if you add a mutation path,
  write it through yourself.
- Icons come from `lucide-react`; several are imported and never used.

## Known issues worth not reintroducing

These are live bugs in the current code. Fix them if a task touches the area;
don't copy the pattern.

- `PantallaHistorial` calls `filtrados.reverse()` and `PantallaDashboard` calls
  `entrenamientos.sort()` — both mutate props/state arrays in place during render.
- `promRPE` uses `parseInt(e.rpe)`, so half-point RPEs (8.5) truncate to 8.
- `mostrarNotificacion` spreads `notificaciones` from the closure instead of
  using the updater form; rapid saves drop toasts.
- `index.html` registers `/sw.js`; that file does not exist, so the service
  worker never installs (the `.catch()` swallows it).
- `TimerModal`'s close button is `absolute` inside a container that is not
  `relative`, so it positions against the viewport.
- `id: Date.now()` is used as both the record id and the React key; two sets
  saved in the same millisecond collide.

## Workflow

- Branch: work on `claude/claude-md-documentation-8jdk6z` unless told otherwise.
  Default branch is `main`.
- There are no tests, no linter run, and no build that currently succeeds —
  verification means reading the diff carefully and, where it matters, standing
  up one of the two lanes above locally.
- Commits in history are single-file and terse (`Create index.js`); prefer
  clearer messages than that.
