# Validate deploy (PROMPT 3)

Sei il Technical Lead del progetto Tobear. Esegui la validazione post-deploy su x10hosting dopo l hardening. Apri questa sessione nella root del super-repo.

## Prerequisiti (devono essere già fatti)

- Backend hardenato, committato, pushato, deployato su x10hosting via GitHub Actions
- Frontend hardenato, committato, pushato, deployato su x10hosting (dist/ in backend/public/app/)
- GitHub secrets creati: APP_KEY, ARTISAN_DEBUG_TOKEN, TEST_CLEANUP_TOKEN, CONTACT_NOTIFICATION_EMAIL, DB_*, MAIL_*, FTP_*
- `php artisan migrate --force` eseguito in produzione (via cPanel/SSH)
- Domain: https://tobear.x10.mx

## Bootstrap (OBBLIGATORIO)

1. Leggi docs/superpowers/reports/AUDIT-2026-07-06.md (audit completo, per contesto).
2. Leggi backend/docs/plans/PLAN-hardening-2026-07-06.md (piano backend eseguito).
3. Leggi frontend/docs/plans/PLAN-hardening-2026-07-06.md (piano frontend eseguito).
4. Leggi gli ultimi handoff di entrambi i repo (backend/handoffs/ e frontend/handoffs/).

## Setup Playwright MCP

Hai il MCP Playwright disponibile (configurato in opencode.json root). Usalo per i test browser-based.

## Validazione — elenco check

Esegui questi check nell ordine. Per ognuno, riporta: comando/azione, risultato ottenuto, risultato atteso, PASS/FAIL.

### A. API e backend (via curl/bash)

A1. `curl -i https://tobear.x10.mx/api/user` (no auth) → atteso 401 (non 500, non CORS error, non HTML redirect)
A2. `curl -i https://tobear.x10.mx/api/v1/contact` con payload valido → atteso 201 o 422 (validazione), non 500
A3. `curl -i https://tobear.x10.mx/serverphpinfo` (no header) → atteso 404
A4. `curl -i -H "X-Debug-Token: wrong" https://tobear.x10.mx/serverphpinfo` → atteso 404
A5. `curl -i -H "X-Debug-Token: <valore-reale>" https://tobear.x10.mx/serverphpinfo` → atteso 200 (chiedimi il token reale prima di questo check)
A6. `curl -i https://tobear.x10.mx/laravelversion` (no header) → atteso 404
A7. `curl -i https://tobear.x10.mx/laravelversion` con token corretto → atteso 200 + JSON con versione Laravel
A8. `curl -i https://tobear.x10.mx/artisan/migrate/status` con token corretto → atteso 200 con stato migrazioni
A9. `curl -i https://tobear.x10.mx/` → atteso 200 + Content-Type text/html (SPA shell)

### B. SPA e asset (via Playwright MCP)

B1. Naviga su https://tobear.x10.mx/ → verifica che carichi la SPA (non pagina Laravel error)
B2. Verifica che `<div id="app">` sia popolato (Vue montato)
B3. Apri DevTools Console → nessun errore CSP (cerca "Content Security Policy" o "Refused")
B4. Apri DevTools Network → verifica che gli asset JS/CSS siano serviti da /app/assets/ con 200 (non 404)
B5. Verifica che `<link rel="manifest">` punti a /app/manifest.webmanifest e sia raggiungibile (200)
B6. Naviga /todo → verifica che la todo page renderizzi (input create visibile)
B7. Naviga /about → verifica about content renderizzi (markdown i18n)
B8. Naviga /login → verifica form login visibile

### C. Auth flow (via Playwright MCP)

C1. Se AUTH_ENABLED=false in prod (default): /login e /register redirect a /todo. Verifica.
C2. Se vuoi testare auth: chiedimi credenziali test. Crea user via /register, verifica email verification flow, login, logout.
C3. Dopo login: verifica che GET /api/user ritorni 200 con JSON utente (non 401)
C4. Dopo login: crea un task via UI, verifica che compaia e persista dopo reload
C5. Logout: verifica che router.push vada a /todo (non errore console per route Home inesistente)

### D. PWA e Service Worker (via Playwright MCP)

D1. Apri Application > Service Workers → verifica sw.js registered e activated
D2. Apri Application > Cache Storage → verifica cache api-cache e static-resources esistano
D3. Verifica che sw.js non contenga ReferenceError su env (apri sw.js source)
D4. Simula offline (Application > Offline) → naviga /todo → deve renderizzare da cache (app shell)
D5. Torna online → verifica che sync riprenda (se ci sono task localOnly)
D6. Apri Application > Manifest → verifica start_url=/app/, scope=/app/, display=standalone, icone presenti

### E. PWABuilder validation

E1. Apri https://www.pwabuilder.com/ e inserisci https://tobear.x10.mx
E2. Esegui il test → riporta score e eventuali issue
E3. Se score < 90 o ci sono errori, documentali

### F. Sicurezza (via curl/bash)

F1. `curl -i https://tobear.x10.mx/api/user` → verifica header Set-Cookie presente (sessione Sanctum)
F2. Verifica header Strict-Transport-Security (se HTTPS forzato dal host)
F3. Verifica header X-Content-Type-Options: nosniff
F4. Verifica che APP_DEBUG=false (nessun stack trace in risposta 500 — triggera con URL invalido tipo /api/v1/nonexistent)
F5. Verifica cookie sessione: Secure flag, HttpOnly flag, SameSite=Lax

### G. Performance (via Playwright MCP)

G1. Apri Performance tab → registra caricamento iniziale di / → riporta: LCP, FCP, TTFB
G2. Verifica che il bundle JS sia servito con Content-Encoding: gzip (Accept-Encoding: gzip)
G3. Verifica dimensione bundle JS iniziale (devi dirmi se > 300KB — indica mancanza lazy loading)

## Output richiesto

Al termine, produci un report strutturato:
Tobear — Validazione Post-Deploy 2026-07-XX
Riepilogo: X PASS / Y FAIL su Z check
Dettaglio per sezione (A-G)
Problemi trovati (con severità)
Raccomandazioni (con priorità)
Prontezza per PWABuilder / Play Store

Se trovi FAIL:

- Per ogni FAIL, proponi la fix (file, riga, diff atteso) ma NON eseguire — presentami il piano e aspetta conferma.
- Se i FAIL sono nel backend, proponi fix da fare in sessione backend separata.
- Se i FAIL sono nel frontend, proponi fix da fare in sessione frontend separata.

Se tutto PASS:

- Conferma che il progetto è pronto per la pubblicazione Play Store (dopo acquisto dominio Namecheap).
- Suggerisci next steps (liste annidate, sync, reels secondo TODO.md).

Alla fine, chiedimi se voglio salvare il report come file (e dove: root del super-repo o Obsidian).

Inizia leggendo i file del bootstrap e conferma che il deploy è stato fatto, poi parti con i check A1-A9.
