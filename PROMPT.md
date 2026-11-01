PROMPT 3 — Sessione Validazione Post-Deploy (apri sessione su todos_app/)
Sei il Technical Lead del progetto Tobear. Esegui la validazione post-deploy su x10hosting dopo l'hardening.

## Prerequisiti (devono essere già fatti)

- Backend hardenato, committato, pushato, deployato su x10hosting via GitHub Actions
- Frontend hardenato, committato, pushato, deployato su x10hosting (dist/ in backend/public/app/)
- GitHub secrets creati: APP*KEY, ARTISAN_DEBUG_TOKEN, TEST_CLEANUP_TOKEN, CONTACT_NOTIFICATION_EMAIL, DB*\_, MAIL\_\_, FTP\_\*
- `php artisan migrate --force` eseguito in produzione (via cPanel/SSH)
- Domain: https://tobear.x10.mx

## Bootstrap (OBBLIGATORIO)

1. Leggi AUDIT-2026-07-06.md (audit completo, per contesto).
2. Leggi backend/docs/plans/PLAN-hardening-2026-07-06.md (piano backend eseguito).
3. Leggi frontend/docs/plans/PLAN-hardening-2026-07-06.md (piano frontend eseguito).
4. Leggi gli ultimi handoff di entrambi i repo (backend/handoffs/ e frontend/handoffs/).

## Setup Playwright MCP

Hai il MCP Playwright disponibile (configurato in opencode.json root). Usalo per i test browser-based.

## Validazione — elenco check

Esegui questi check nell'ordine. Per ognuno, riporta: comando/azione, risultato ottenuto, risultato atteso, PASS/FAIL.

### A. API e backend (via curl/bash)

A1. `curl -i https://tobear.x10.mx/api/user` (no auth) → atteso 401 (non 500, non CORS error, non HTML redirect)
A2. `curl -i https://tobear.x10.mx/api/v1/contact` con payload valido → atteso 201 o 422 (validazione), non 500
A3. `curl -i https://tobear.x10.mx/serverphpinfo` (no header) → atteso 404
A4. `curl -i -H "X-Debug-Token: wrong" https://tobear.x10.mx/serverphpinfo` → atteso 404
A5. `curl -i -H "X-Debug-Token: <valore-reale>" https://tobear.x10.mx/serverphpinfo` → atteso 200 (chiedimi il token reale prima di questo check)
A6. `curl -i https://tobear.x10.mx/laravelversion` (no header) → atteso 404
A7. `curl -i https://tobear.x10.mx/laravelversion` con token corretto → atteso 200 + JSON `{"Laravel":"12.x.x"}`
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
C5. Logout: verifica che router.push vada a /todo (non errore console per route 'Home' inesistente)

### D. PWA e Service Worker (via Playwright MCP)

D1. Apri Application > Service Workers → verifica sw.js registered e activated
D2. Apri Application > Cache Storage → verifica cache 'api-cache' e 'static-resources' esistano
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
G2. Verifica che il bundle JS sia servito con Content-Encoding: gzip ( Accept-Encoding: gzip)
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

Alla fine, chiedimi se voglio salvare il report come file (e dove: root todos_app/ o Obsidian).

Inizia leggendo i file del bootstrap e conferma che il deploy è stato fatto, poi parti con i check A1-A9.

---

PROMPT 6 — Sessione Re-Validazione Post-Fix (apri sessione su todos_app/)
Sei il Technical Lead del progetto Tobear. Esegui la re-validazione post-fix su x10hosting dopo l'applicazione dei fix backend (PROMPT 4) e frontend (PROMPT 5).

## Prerequisiti (devono essere già fatti)

- Fix backend eseguiti, committati, pushati, deployati su x10hosting via GitHub Actions (PROMPT 4)
- Fix frontend eseguiti, committati, pushati, deployati su x10hosting (PROMPT 5)
- `php artisan config:clear` (o cancellazione di bootstrap/cache/config.php) eseguito in produzione dopo i fix
- Domain: https://tobear.x10.mx

## Bootstrap (OBBLIGATORIO)

1. Leggi VALIDATION-2026-07-07.md (report validazione originale — per contesto sui FAIL/WARN trovati).
2. Leggi backend/docs/plans/PLAN-validation-fixes-2026-07-07.md (piano backend eseguito).
3. Leggi frontend/docs/plans/PLAN-validation-fixes-2026-07-07.md (piano frontend eseguito).
4. Leggi gli ultimi handoff di entrambi i repo (backend/handoffs/ e frontend/handoffs/).

## Setup Playwright MCP

Hai il MCP Playwright disponibile (configurato in opencode.json root). Usalo per i test browser-based.

## Re-validazione — elenco check mirati

Esegui SOLO questi check. Non re-eseguire l'intero PROMPT 3 — gran parte dei check già PASSavano e non sono impattati dai fix.

Per ogni check, riporta: comando/azione, risultato ottenuto, risultato atteso, PASS/FAIL.

### A. Fix backend verificati (via curl/bash)

| # | Comando | Atteso | Note |
|---|---|---|---|
| A3-bis | `curl -i https://tobear.x10.mx/serverphpinfo` (no header) | 404 | Regressione: fail-closed deve ancora funzionare |
| A5-bis | `curl -i -H "X-Debug-Token: <valore-reale>" /serverphpinfo` | 200 | Chiedimi il token reale |
| A6-bis | `curl -i https://tobear.x10.mx/laravelversion` (no header) | 404 | Regressione |
| A7-bis | `curl -i -H "X-Debug-Token: <TOKEN>" /laravelversion` | 200 + `{"Laravel":"12.15.0"}` | |
| A9-bis | `curl -i https://tobear.x10.mx/` → grep `manifest` | 200 + HTML con `<link rel="manifest">` | **IMPORTANTE**: verifica che il fix cspDynamicPlugin abbia ripristinato il tag manifest e viewport |
| A10 | `curl -i https://tobear.x10.mx/app/todo` | 200 + text/html SPA shell (NON 404) | **NUOVO**: verifica che Route::fallback matchi path multi-segment |
| A11 | `curl -i https://tobear.x10.mx/app/about` | 200 + text/html SPA shell | Stesso, altro path |
| F2-bis | `curl -i https://tobear.x10.mx/` → grep `Strict-Transport-Security` | Header presente (o assente se .htaccess non supportato) | |
| F3-bis | `curl -i https://tobear.x10.mx/` → grep `X-Content-Type-Options` | `nosniff` presente (o assente) | |
| F5-bis | `curl -i -H "Accept: application/json" /api/user` → grep `domain=` | `tobear.x10.mx` se fix cookie domain funzionato | |

### B. Fix frontend verificati (via Playwright MCP + curl)

B1. Naviga su https://tobear.x10.mx/ → verifica SPA caricata.
B2. `<div id="app">` popolato (Vue montato).
B3. Console → nessun errore CSP (cerca "Content Security Policy" o "Refused").
B4. Asset JS/CSS serviti da /app/assets/ con 200.
B5. **KEY**: verifica che `<link rel="manifest">` sia presente nell'HTML e punti al file raggiungibile:
   - Usa `document.querySelector('link[rel="manifest"]')` → non deve essere null
   - Fetch dell'href → deve rispondere 200 + content-type `application/manifest+json`
B6. Naviga `/app/todo` → todo page render, input create visibile (via SPA o URL diretto).
B7. Naviga `/app/about` → about content renderizzato (markdown i18n).
B8. Naviga `/app/login` → redirect a `/app/todo` (AUTH_ENABLED=false).

### C. PWA e Service Worker (via Playwright MCP)

D1. SW registration: verificare registered + activated.
D2. Cache Storage: `workbox-precache-v2` + eventuali runtime caches.
D3. sw.js source: nessun ReferenceError su env (verificato, ma DA CONFERMARE che il fix non sia stato sovrascritto).
D6. **KEY**: Apri Application > Manifest (o equivalente via evaluate):
   - `start_url` deve essere `/app/`
   - `scope` deve essere `/app/`
   - `display` deve essere `standalone`
   - Icone 192x192 e 512x512 devono essere presenti e raggiungibili
   - `theme_color` deve essere `#8b5e3c`

### D. PWABuilder validation (via Playwright MCP) — 🔴 KEY CHECK

D1. Apri https://www.pwabuilder.com/ e inserisci https://tobear.x10.mx
D2. Esegui il test → riporta score e issue list
D3. **Atteso**: score ≥ 90. Se score < 90, documenta le issue.
D4. Se score ≥ 90, prova a generare il package per Android (Play Store).

### E. Performance (via curl)

G2. Content-Encoding: gzip su asset JS → deve essere gzip.
G3. **KEY**: dimensione bundle JS iniziale < 300KB (verifica con `curl -sS -D - /app/assets/index-*.js | grep content-length`). Se ancora > 300KB, indica che il lazy loading non è stato applicato correttamente.

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

Confronto con VALIDATION-2026-07-07.md:
- FAIL risolti: [elenco]
- FAIL rimanenti: [elenco] (se presenti)
- Nuovi problemi: [elenco] (se presenti)

Prontezza per PWABuilder / Play Store: [SI/NO]

Se tutto PASS:
- Conferma che il progetto è pronto per la pubblicazione Play Store (dopo acquisto dominio Namecheap).
- Suggerisci next steps (liste annidate, sync, reels secondo TODO.md).

Alla fine, chiedimi se voglio salvare il report come file.

Inizia leggendo i file del bootstrap e chiedimi il token reale per A5-bis prima di partire.

---

# PROMPT 7 — Ottimizzazione Performance Post-Lighthouse

Sei il Technical Lead del progetto Tobear. Implementa le ottimizzazioni emerse dal report Lighthouse reale (Chrome DevTools, mobile emulated 4x CPU) per portare il Performance score da 50 a ≥ 85.

## Contesto

- **Dominio**: https://tobear.x10.mx
- **Repo**: `todos_app/frontend/` (Vue 3 + Vite + vite-plugin-pwa)
- **Lighthouse reale mobile**: Performance 50, LCP 4.2s, TBT 2.160ms, FCP 2.4s, CLS 0
- **Causa principale**: bundle JS 511KB (index.js 320KB + TodoPage.js 191KB) su CPU 4x = 2.5s script evaluation
- **Server**: x10hosting LiteSpeed con **modulo Brotli attivo** (PHP + server-level)
- **Compressione attuale**: solo gzip via `vite-plugin-compression` (114KB per il bundle)

## Bootstrap (OBBLIGATORIO)

1. Leggi `LH-REPORT-2026-07-07.md` (root `todos_app/`) — analisi completa Lighthouse con opportunità
2. Leggi `RE-VALIDATION-2026-07-07.md` (root `todos_app/`) — stato attuale post-fix
3. Leggi `frontend/AGENTS.md` — convenzioni progetto
4. Leggi `frontend/vite.config.js` — config build attuale (VitePWA, viteCompression gzip, cspDynamicPlugin)
5. Leggi `frontend/index.html` — template HTML (canonical `%VITE_BASE_URL%`, registerSW.js, tag manifest/icon)
6. Leggi `frontend/src/router/index.js` — router con lazy loading già applicato
7. Leggi `frontend/src/pages/TodoPage.vue` — pagina più pesante (191KB chunk)
8. Leggi `frontend/public/.htaccess` — SPA fallback + MIME webmanifest
9. Leggi `backend/public/.htaccess` — security headers + HSTS + cookie domain
10. Leggi `frontend/package.json` — dipendenze (vite-plugin-pwa, vite-plugin-compression, vuedraggable, ecc.)

## Task da implementare

### P1 — Performance (impatto alto, target score 75-85)

#### 1. Code splitting più aggressivo (`vite.config.js`)

Configurare `build.rollupOptions.output.manualChunks` per separare:
- `vendor-vue`: vue, vue-router, pinia (core framework ~80KB)
- `vendor-ui`: vuedraggable, @headlessui/vue (UI libs ~30KB)
- `vendor-utils`: axios, idb, markdown-it (utilities ~20KB)
- `index`: app code residuo (deve scendere da 320KB a < 150KB)

**Verifica**: dopo build, `dist/assets/index-*.js` deve essere < 150KB. I chunk vendor devono essere caricati in parallelo.

**Nota**: NON spostare TodoPage nel bundle principale — resta lazy. Valuta se spostare `useGuestMigration` (5KB, usato solo in certi flow) fuori dal main chunk.

#### 2. Defer registerSW.js (`index.html`)

Attualmente `<script src="registerSW.js" defer></script>` — il `defer` non è sufficiente perché è uno script blocking nel head. Spostarlo in fondo al `<body>` oppure caricarlo post-load via `requestIdleCallback`:

```html
<script>
  window.addEventListener('load', () => {
    requestIdleCallback(() => {
      import('./registerSW.js').then(m => m.registerSW?.()).catch(() => {})
    })
  })
</script>
```

**Verifica**: registerSW.js non deve apparire nei render-blocking resources di Lighthouse.

#### 3. fetchpriority="high" sul logo LCP (`PwaInstallBanner.vue` o `MobileNavFab.vue`)

Il LCP element è `img/icons/icon-128x128.png` (visualizzata a 56x56px). Aggiungere:

```html
<img src="..." alt="toBear" fetchpriority="high" />
```

**Verifica**: LCP breakdown `resourceLoadDelay` deve scendere da 2.5s a < 1s.

#### 4. Brotli compression (server + build)

**4a. `.htaccess`** (entrambi `frontend/public/.htaccess` e `backend/public/.htaccess`):

```apache
# Brotli compression (LiteSpeed/Apache)
<IfModule mod_brotli.c>
  AddOutputFilterByType BROTLI_COMPRESS text/html text/plain text/css text/javascript application/javascript application/json application/manifest+json image/svg+xml
</IfModule>
<IfModule lsmod_brotli.c>
  AddOutputFilterByType BROTLI_COMPRESS text/html text/plain text/css text/javascript application/javascript application/json application/manifest+json image/svg+xml
</IfModule>
```

**4b. `vite-plugin-compression`** (`vite.config.js`):

Aggiungere una seconda istanza del plugin per generare `.br` files pre-compressi:

```js
import viteCompression from 'vite-plugin-compression'

viteCompression({
  verbose: true,
  disable: false,
  threshold: 1025,
  algorithm: 'brotliCompress',
  ext: '.br',
}),
```

**4c. `.htaccess` per servire .br files** (se server non comprime on-the-fly):

```apache
<IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteCond %{HTTP:Accept-Encoding} br
  RewriteCond %{REQUEST_FILENAME}.br -f
  RewriteRule ^(.*)$ $1.br [L]
  RewriteCond %{HTTP:Accept-Encoding} gzip
  RewriteCond %{REQUEST_FILENAME}.gz -f
  RewriteRule ^(.*)$ $1.gz [L]
</IfModule>
<IfModule mod_headers.c>
  <FilesMatch "\.(js|css)$">
    Header append Vary Accept-Encoding
  </FilesMatch>
</IfModule>
```

**Verifica**: `curl -sH "Accept-Encoding: br" -I https://tobear.x10.mx/app/assets/index-*.js` → `content-encoding: br`. Transfer size deve scendere da 114KB a ~90KB.

#### 5. Inline CSS critico o preload (`index.html`)

Il CSS principale (9KB gzip) è render-blocking. Opzione consigliata:

```html
<link rel="preload" href="assets/index-CCmZ2gYx.css" as="style" onload="this.onload=null;this.rel='stylesheet'">
<noscript><link rel="stylesheet" href="assets/index-CCmZ2gYx.css"></noscript>
```

**Verifica**: CSS non deve apparire nei render-blocking resources.

### P2 — Accessibility + SEO (impatto medio)

#### 6. Fix color contrast (CSS / design system)

Dal report Lighthouse, 2 violazioni:

**6a. PWA Install Banner**:
- Testo `#6b6860` su background `#2e2c28` → ratio 2.5:1 (serve ≥ 4.5:1)
- Fix: schiarire il testo a `#9a958a` o simile

**6b. Bottone "Install"**:
- `#ffffff` su `bg-tb-accent` `#e8823a` → ratio 2.73:1 (serve ≥ 4.5:1)
- Fix: scurire `bg-tb-accent` a `#c4682a`

**Verifica**: Lighthouse Accessibility score deve salire da 90 a ≥ 95.

#### 7. Fix struttura lista `<li>` senza `<ul>` (`TodoPage.vue`)

I task items sono `<li>` dentro un `<div>` (vuedraggable). Lighthouse segnala `listitem: FAIL`.

Fix: aggiungere `role="list"` al container di vuedraggable:

```html
<div role="list" class="space-y-2">
  <draggable :list="tasks" ...>
    <li v-for="task in tasks" ...>...</li>
  </draggable>
</div>
```

**Verifica**: Lighthouse audit `listitem` deve passare.

#### 8. Fix canonical URL (`index.html` riga 20)

Attualmente: `<link rel="canonical" href="%VITE_BASE_URL%" />` — non sostituito da Vite.

Fix: sostituire con URL assoluto (aggiornare al nuovo dominio quando disponibile):

```html
<link rel="canonical" href="https://tobear.x10.mx/app/" />
```

**Verifica**: Lighthouse SEO audit `canonical` deve passare (URL assoluto valido).

### P3 — Bonus (se tempo permette)

#### 9. Preconnect per Matomo (`index.html`)

```html
<link rel="preconnect" href="https://stats.tobear.x10.mx" />
```

#### 10. Target esnext per rimuovere legacy JS (`vite.config.js`)

```js
build: {
  target: 'esnext',
  // ...
}
```

Rimuove ~19KB di polyfills Babel non necessari per browser moderni.

## Workflow

1. Per ogni task, modifica il file, verifica con `npm run build` e `npm run lint`
2. Testa localmente con `npm run dev` se possibile
3. Commit con Conventional Commits EN
4. **NON fare git push** — lo fa l'utente dopo verifica
5. Scrivi handoff in `frontend/handoffs/2026-07-07-performance-optimization.md`

## Verifica post-build (locale)

```bash
# Verifica chunk sizes
ls -lh dist/assets/*.js
# index-*.js deve essere < 150KB (era 320KB)
# vendor-*-*.js devono esistere

# Verifica .br files generati
ls -lh dist/assets/*.br

# Verifica index.html
grep 'canonical' dist/index.html       # href assoluto
grep 'fetchpriority' dist/index.html   # fetchpriority="high"

# Verifica .htaccess
grep -i brotli dist/.htaccess
```

## Verifica post-deploy (su x10hosting)

```bash
# Brotli attivo
curl -sH "Accept-Encoding: br" -I https://tobear.x10.mx/app/assets/index-*.js | grep content-encoding

# Transfer size ridotta
curl -sH "Accept-Encoding: br" -o /dev/null -w "%{size_download}" https://tobear.x10.mx/app/assets/index-*.js

# Canonical assoluto
curl -s https://tobear.x10.mx/ | grep canonical
```

## Output richiesto

1. **Handoff** in `frontend/handoffs/2026-07-07-performance-optimization.md`
2. **Riepilogo modifiche** con file toccati e chunk sizes before/after
3. **Stima impatto** su score Lighthouse (target: Performance 75-85, Accessibility 95+, SEO 100)
4. **Commit** (no push) con messaggio:
   ```
   perf: optimize bundle splitting, compression, and a11y

   - Add manualChunks for vendor-vue, vendor-ui, vendor-utils
   - Enable Brotli compression (server .htaccess + vite-plugin-compression)
   - Defer registerSW.js to reduce render-blocking
   - Add fetchpriority=high on LCP image
   - Fix color contrast on Install button and PWA banner
   - Fix <li> list structure for vuedraggable items
   - Fix canonical URL to absolute https://tobear.x10.mx/app/
   - Add preconnect for Matomo stats domain
   ```

## File rilevanti

```
frontend/vite.config.js                    # manualChunks + brotli + target esnext
frontend/index.html                        # canonical, registerSW defer, preconnect, fetchpriority
frontend/public/.htaccess                  # brotli config + serve .br/.gz
backend/public/.htaccess                   # brotli config
frontend/src/pages/TodoPage.vue            # list structure role="list"
frontend/src/components/PwaInstallBanner.vue # color contrast + fetchpriority
frontend/src/components/MobileNavFab.vue    # fetchpriority + color contrast
frontend/src/assets/themes.css             # bg-tb-accent contrast fix (se serve)
```

---

# PROMPT 8 — Verifica Lighthouse Post-Ottimizzazione Performance

Sei il Technical Lead del progetto Tobear. Verifica l'impatto reale delle ottimizzazioni performance implementate nella sessione PROMPT 7 (performata sul commit `85afa31` su `tobear-vuejs-frontend`, deployato su x10hosting).

## Contesto

- **Dominio**: https://tobear.x10.mx
- **Repo**: `todos_app/frontend/` (Vue 3 + Vite + vite-plugin-pwa)
- **Sessione precedente**: `frontend/handoffs/2026-07-07-performance-optimization.md`
- **Baseline pre-fix (Lighthouse mobile reale, 4x CPU)**:
  - Performance **50**, LCP 4.2s, TBT 2.160ms, FCP 2.4s, CLS 0
  - Accessibility 90, Best Practices 92, SEO 92
- **Ottimizzazioni applicate**:
  - manualChunks: index.js 313KB → 53KB (vendor 438KB singolo chunk)
  - Brotli on-the-fly via .htaccess (AddOutputFilterByType BROTLI_COMPRESS)
  - CSS non render-blocking (media="print" + onload trick)
  - fetchpriority="high" su logo LCP
  - target esnext (polyfills Babel rimossi)
  - Fix SW navigateFallback subpath (`/index.html` → `index.html`)
  - Fix color contrast (accent #e8823a → #c4682a, banner text)
  - Fix listitem role="list" su vuedraggable
  - Canonical assoluto https://tobear.x10.mx/app/
  - Preconnect Matomo

## Bootstrap (OBBLIGATORIO)

1. Leggi `frontend/AGENTS.md` (convenzioni + regola no-amend).
2. Leggi `frontend/handoffs/2026-07-07-performance-optimization.md` (sessione precedente).
3. Leggi `frontend/vite.config.js` (config build attuale).
4. Solo dopo, esegui le verifiche.

## Task

### 1. Lighthouse mobile reale (Chrome DevTools, 4x CPU slowdown)

Esegui Lighthouse su `https://tobear.x10.mx/app/todo` con emulazione mobile (moto g power, Android 11, 4x CPU slowdown). Confronta con la baseline:

| Metrica | Baseline (pre-fix) | Target |
|---|---|---|
| Performance | 50 | **≥ 80** |
| LCP | 4.2s | **< 2.5s** |
| TBT | 2.160ms | **< 600ms** |
| FCP | 2.4s | **< 1.8s** |
| CLS | 0 | **0** |
| Accessibility | 90 | **≥ 95** |
| SEO | 92 | **100** |
| Best Practices | 92 | **≥ 95** |

### 2. Verifica compressioni e MIME type (post-fix brotli on-the-fly)

```bash
# Brotli attivo con Content-Type CORRETTO
curl -sH "Accept-Encoding: br" -I https://tobear.x10.mx/app/assets/index-*.js
# Atteso: content-type: application/javascript  +  content-encoding: br

# CSS
curl -sH "Accept-Encoding: br" -I https://tobear.x10.mx/app/assets/index-*.css
# Atteso: content-type: text/css  +  content-encoding: br

# Transfer size
curl -sH "Accept-Encoding: br" -o /dev/null -w "%{size_download}\n" \
  https://tobear.x10.mx/app/assets/vendor-*.js
# Atteso: ~138KB
```

**REGRESSION CHECK**: se `content-type` è `application/octet-stream`, il brotli on-the-fly non funziona — investigare alternativa.

### 3. Verifica chunk sizes via DevTools Network panel

| File | Baseline (gzip) | Target (brotli) |
|---|---|---|
| index.js | 114KB | **~13KB** |
| vendor.js | (in index, 320KB) | **~138KB** |
| TodoPage.js (lazy) | 66KB | **~4.5KB** |
| CSS | 9.3KB | **~8KB** |

### 4. Audit Lighthouse specifici (checklist)

- [ ] `render-blocking-resources`: CSS NON deve apparire (media="print" trick)
- [ ] `unused-javascript`: index.js < 50KB unused (era 50KB secondo report)
- [ ] `legacy-javascript`: nessun polyfill Babel (target esnext)
- [ ] `uses-rel-preconnect`: Matomo preconnect presente
- [ ] `canonical`: URL assoluto https://tobear.x10.mx/app/
- [ ] `color-contrast`: nessuna violazione
- [ ] `listitem` (a11y): PASS (role="list" su vuedraggable)

### 5. Verifica PWA / Service Worker

```bash
# SW raggiungibile
curl -s -o /dev/null -w "SW: %{http_code}" https://tobear.x10.mx/app/sw.js

# SPA fallback (navigation)
curl -s -o /dev/null -w "%{http_code}" https://tobear.x10.mx/app/sconosciuta
# Atteso: 200 (SPA shell, non 404)
```

### 6. Esegui E2E tests

```bash
cd frontend
npm run test:e2e:pwa
# Atteso: 3 passed, 1 skipped (test.fixme offline)
```

### 7. Documenta risultati

Scrivi `frontend/handoffs/2026-07-08-lighthouse-verification.md` con:
- Screenshot Lighthouse (mobile reale 4x CPU) — se hai Playwright MCP, cattura uno screenshot della pagina
- Tabella metriche before/after/target
- Audit pass/fail
- Eventuali regressioni o problemi residui
- Raccomandazioni per ulteriori ottimizzazioni (se score < 85)

### 8. Se score < 85 (opzionale, solo se sotto target)

Identifica colli bottiglia residui:
- `unused-javascript` > 20KB → ulteriori split o lazy loading
- LCP > 2.5s → preload hero image o riduci vendor chunk
- TBT > 600ms → valuta code splitting di vendor chunk (vue-i18n è pesante?)
- Dominant third-party → Matomo analytics (carica async, già configurato)

## Output richiesto

1. **Handoff** in `frontend/handoffs/2026-07-08-lighthouse-verification.md`
2. **Riepilogo score** in tabella before/after/target
3. **Eventuale commit** (no --amend, no push) per fix residui (se score < 85)

## Regole operative

- NON usare `git commit --amend` — vedi AGENTS.md per la regola.
- NON eseguire `git push` senza richiesta esplicita.
```
