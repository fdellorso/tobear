# toBear — super-repo

Todo app in stile Clear (iOS/web): backend Laravel + frontend Vue PWA offline-first.

## Struttura

- `backend/` — submodule `tobear-laravel-backend` (API REST, Sanctum cookie-based)
- `frontend/` — submodule `tobear-vuejs-frontend` (Vue 3 + Vite + Tailwind, PWA)
- File gestione in root: `AGENTS.md`, `opencode.json` (config root), comandi in `.opencode/commands/`
- Report/audit in `docs/superpowers/reports/`: `AUDIT-*`, `VALIDATION-*`,
  `RE-VALIDATION-*`, `LH-REPORT-*` (+ allegati pwabuilder/snapshot)

## Clone

```bash
git clone --recurse-submodules https://github.com/fdellorso/tobear.git
```

Chi ha già clonato senza submodule:

```bash
git submodule update --init --recursive
```

## Note

- I submodule sono pinnati ai commit verificati; per aggiornarli:
  `git submodule update --remote backend frontend` (poi commit del nuovo pin).
- Password nei report archiviati (`docs/superpowers/reports/`) = solo sviluppo, mai produzione.
