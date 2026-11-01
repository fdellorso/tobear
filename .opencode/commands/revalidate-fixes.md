# Revalidate fixes (PROMPT 6)

Sei il Technical Lead del progetto Tobear. Esegui la re-validazione post-fix su x10hosting dopo l applicazione dei fix backend e frontend. Apri questa sessione nella root del super-repo.

## Prerequisiti (devono essere già fatti)

- Fix backend eseguiti, committati, pushati, deployati su x10hosting via GitHub Actions
- Fix frontend eseguiti, committati, pushati, deployati su x10hosting
- `php artisan config:clear` (o cancellazione di bootstrap/cache/config.php) eseguito in produzione dopo i fix
- Domain: https://tobear.x10.mx

## Bootstrap (OBBLIGATORIO)

1. Leggi docs/superpowers/reports/VALIDATION-2026-07-07.md (report validazione originale — per contesto sui FAIL/WARN trovati).
2. Leggi backend/docs/plans/PLAN-validation-fixes-2026-07-07.md (piano backend eseguito).
3. Leggi frontend/docs/plans/PLAN-validation-fixes-2026-07-07.md (piano frontend eseguito).
4. Leggi gli ultimi handoff di entrambi i repo (backend/handoffs/ e frontend/handoffs/).

## Setup Playwright MCP

Hai il MCP Playwright disponibile (configurato in opencode.json root). Usalo per i test browser-based.

## Re-validazione — elenco check mirati

Esegui SOLO questi check. Non re-eseguire l intero validate-deploy — gran parte dei check già PASSavano e non sono impattati dai fix.

Per ogni check, riporta: comando/azione, risultato ottenuto, risultato atteso, PASS/FAIL.

### A. Fix backend verificati (via curl/bash)

| # | Comando | Atteso | Note |
|---|---|---|---|
| A3-bis | `curl -i https://tobear.x10.mx/serverphpinfo` (no header) | 404 | Regressione: fail-closed deve ancora funzionare |
| A5-bis | `curl -i -H "X-Debug-Token: <valore-reale>" /serverphpinfo` | 200 | Chiedimi il token reale |
| A6-bis | `curl -i https://tobear.x10.mx/laravelversion` (no header) | 404 | Regressione |
| A7-bis | `curl -i -H "X-Debug-Token: <TOKEN>" /laravelversion` | 200 + JSON versione Laravel | |
| A9-bis | `curl -i https://tobear.x10.mx/` → grep `manifest` | 200 + HTML con `<link rel="manifest">` | IMPORTANTE: verifica che il fix cspDynamicPlugin abbia ripristinato il tag manifest e viewport |
| A10 | `curl -i https://tobear.x10.mx/app/todo` | 200 + text/html SPA shell (NON 404) | NUOVO: verifica che Route::fallback matchi path multi-segment |
| A11 | `curl -i https://tobear.x10.mx/app/about` | 200 + text/html SPA shell | Stesso, altro path |
| F2-bis | `curl -i https://tobear.x10.mx/` → grep `Strict-Transport-Security` | Header presente (o assente se .htaccess non supportato) | |
| F3-bis | `curl -i https://tobear.x10.mx/` → grep `X-Content-Type-Options` | `nosniff` presente (o assente) | |
| F5-bis | `curl -i -H "Accept: application/json" /api/user` → grep `domain=` | `tobear.x10.mx` se fix cookie domain funzionato | |

### B. Fix frontend verificati (via Playwright MCP + curl)

B1. Naviga su https://tobear.x10.mx/ → verifica SPA caricata.
B2. `<div id="app">` popolato (Vue montato).
B3. Console → nessun errore CSP (cerca "Content Security Policy" o "Refused").
B4. Asset JS/CSS serviti da /app/assets/ con 200.
B5. KEY: verifica che `<link rel="manifest">` sia presente nell HTML e punti al file raggiungibile:
   - Usa `document.querySelector('link[rel="manifest"]')` → non deve essere null
   - Fetch dell href → deve rispondere 200 + content-type `application/manifest+json`
B6. Naviga `/app/todo` → todo page render, input create visibile (via SPA o URL diretto).
B7. Naviga `/app/about` → about content renderizzato (markdown i18n).
B8. Naviga `/app/login` → redirect a `/app/todo` (AUTH_ENABLED=false).

### C. PWA e Service Worker (via Playwright MCP)

D1. SW registration: verificare registered + activated.
D2. Cache Storage: `workbox-precache-v2` + eventuali runtime caches.
D3. sw.js source: nessun ReferenceError su env (verificato, ma DA CONFERMARE che il fix non sia stato sovrascritto).
D6. KEY: Apri Application > Manifest (o equivalente via evaluate):
   - `start_url` deve essere `/app/`
   - `scope` deve essere `/app/`
   - `display` deve essere `standalone`
   - Icone 192x192 e 512x512 devono essere presenti e raggiungibili
   - `theme_color` deve essere `#8b5e3c`

### D. PWABuilder validation (via Playwright MCP) — KEY CHECK

D1. Apri https://www.pwabuilder.com/ e inserisci https://tobear.x10.mx
D2. Esegui il test → riporta score e issue list
D3. Atteso: score ≥ 90. Se score < 90, documenta le issue.
D4. Se score ≥ 90, prova a generare il package per Android (Play Store).

### E. Performance (via curl)

G2. Content-Encoding: gzip su asset JS → deve essere gzip.
G3. KEY: dimensione bundle JS iniziale < 300KB (verifica con curl). Se ancora > 300KB, indica che il lazy loading non è stato applicato correttamente.

### F. Opzionale (se vuoi)

- D4. Simula offline → naviga / → SPA da cache (app shell). Richiede SW controllante.
- G1. Performance tab → registra LCP, FCP, TTFB per benchmark.

## Output richiesto

Al termine, produci un report strutturato breve:

```
Tobear — Re-Validazione Post-Fix 2026-07-XX
Fix backend applicati: [OK/FAIL]
Fix frontend applicati: [OK/FAIL]
PWABuilder score: XX/100 [PASS/FAIL]
Bundle size: XXX KB [PASS/FAIL (> 300KB)]
Catch-all multi-segment: [PASS/FAIL]
HTML completo (manifest/viewport/favicon): [PASS/FAIL]
```

Confronto con docs/superpowers/reports/VALIDATION-2026-07-07.md:
- FAIL risolti: [elenco]
- FAIL rimanenti: [elenco] (se presenti)
- Nuovi problemi: [elenco] (se presenti)

Prontezza per PWABuilder / Play Store: [SI/NO]

Se tutto PASS:
- Conferma che il progetto è pronto per la pubblicazione Play Store (dopo acquisto dominio Namecheap).
- Suggerisci next steps (liste annidate, sync, reels secondo TODO.md).

Alla fine, chiedimi se voglio salvare il report come file.

Inizia leggendo i file del bootstrap e chiedimi il token reale per A5-bis prima di partire.
