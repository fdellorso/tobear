# toBear — Root (super-repo)

Super-repo di coordinamento: `backend/` e `frontend/` sono **submodule git**
(repo PUBLIC separati). Qui vivono solo file gestione (PROMPT, audit/validation,
report), config opencode root e README.

## Dove aprire le sessioni

- Lavoro su API/logica → apri opencode in `backend/` (legge il suo AGENTS.md).
- Lavoro su UI/PWA → apri opencode in `frontend/` (legge il suo AGENTS.md).
- Solo sessioni trasversali qui in root: validazione post-deploy (PROMPT 3),
  coordinamento be+fe, release. Mai codice da una sessione root: se serve,
  apri la sessione nel submodule giusto.

## Bootstrap sessione root

All'inizio di OGNI sessione root, in quest'ordine:

1. Leggi questo AGENTS.md per intero.
2. Leggi `PROMPT.md` per intero.
3. Leggi gli ultimi 2 file in `backend/handoffs/` e `frontend/handoffs/`
   (ordinati per data, il più recente per primo).
4. Solo dopo, chiedi cosa si vuole fare oggi.

## Regole submodule (NON improvvisare)

- Working tree be/fe intoccabili da qui per ristrutturazioni: niente `mv`,
  niente spostamenti di dir — i submodule puntano a path fissi.
- Per aggiornare il pin: `git submodule update --remote backend frontend`,
  verifica, poi commit del nuovo pin in root (mai commit del pin senza verifica).
- Clone sempre con `git clone --recurse-submodules` (vedi README.md).
- `.env`, `vendor/`, `node_modules/`, `*.sqlite` restano locali ai submodule,
  mai nel super-repo.

## Dettagli (non duplicati qui)

- Convenzioni API/auth/pattern → `backend/AGENTS.md`.
- Convenzioni Vue/offline/PWA → `frontend/AGENTS.md`.
- Piano validazione post-deploy → `PROMPT.md` (PROMPT 3).
- Password nei doc root = solo sviluppo, mai produzione.
