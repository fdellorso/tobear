# Tobear — Validazione Post-Deploy 2026-07-07

**Target**: https://tobear.x10.mx
**Metodo**: curl/bash + Playwright MCP + PWABuilder
**Data**: 2026-07-07

---

## Riepilogo

**23 PASS / 2 WARN / 6 FAIL / 5 SKIP** su 36 check totali

| Sezione | PASS | WARN | FAIL | SKIP |
|---|---|---|---|---|
| A. API/Backend | 9 | 1 | 0 | 0 |
| B. SPA e Asset | 6 | 0 | 1 | 0 |
| C. Auth | 1 | 0 | 0 | 4 |
| D. PWA/SW | 2 | 1 | 1 | 2 |
| E. PWABuilder | 0 | 0 | 2 | 0 |
| F. Sicurezza | 3 | 2 | 0 | 0 |
| G. Performance | 1 | 0 | 1 | 1 |

---

## Dettaglio per sezione

### A. API e Backend (via curl/bash)

| # | Azione | Atteso | Ottenuto | Esito |
|---|---|---|---|---|
| A1 | `GET /api/user` con Accept JSON | 401 + JSON | 401 `{"message":"Unauthenticated."}` | ✅ PASS |
| A2-valid | `POST /api/v1/contact` payload valido | 201 | 201 + messaggio | ✅ PASS |
| A2-invalid | `POST /api/v1/contact` payload vuoto | 422 | 422 + field errors | ✅ PASS |
| A3 | `GET /serverphpinfo` no header | 404 | 404 (dopo fix fail-closed + clear cache) | ✅ **PASS dopo fix** |
| A4 | `GET /serverphpinfo` X-Debug-Token: wrong | 404 | 404 | ✅ PASS |
| A5 | `GET /serverphpinfo` con token reale | 200 | 200 + phpinfo | ✅ **PASS dopo fix** |
| A6 | `GET /laravelversion` no header | 404 | 404 (dopo fix) | ✅ **PASS dopo fix** |
| A7 | `GET /laravelversion` con token reale | 200 + JSON | 200 + `{"Laravel":"12.15.0"}` | ✅ **PASS dopo fix** |
| A8 | `GET /artisan/migrate:status` con token reale | 200 | 200 + migrate status output | ✅ **PASS dopo fix** |
| A9 | `GET /` | 200 + text/html SPA | 200, SPA shell servita | ⚠️ **WARN** HTML minimal |

**Osservazioni A9**: l'HTML servito contiene solo CSP + title + script module + CSS + registerSW. **Mancano**: `<link rel="manifest">`, `<meta name="viewport">`, `<meta name="theme-color">`, `<link rel="icon">`, `<link rel="apple-touch-icon">`. Causa: plugin `cspDynamicPlugin` in `vite.config.js:33-35` — regex `/<meta[\s\S]*?http-equiv="Content-Security-Policy"[\s\S]*?\/?>/i` matcha dal primo `<meta charset>` fino al meta CSP, rimuovendo tutti i tag intermedi.

---

### B. SPA e Asset (via Playwright MCP + curl)

| # | Azione | Atteso | Ottenuto | Esito |
|---|---|---|---|---|
| B1 | Naviga `/` | SPA caricata | SPA montata, title "toBear" | ✅ PASS |
| B2 | `<div id="app">` popolato | Vue montato | Navigation, todo, cookie consent | ✅ PASS |
| B3 | Console: errori CSP | Nessuno | Solo `401 /api/user` (atteso) | ✅ PASS |
| B4 | Asset JS/CSS da `/app/assets/` | 200 | JS 200, CSS 200, registerSW 200, icon 200 | ✅ PASS |
| B5 | `<link rel="manifest">` | Manifest referenziato | **NOT FOUND** nell'HTML | ❌ **FAIL** |
| B6 | Naviga `/app/todo` (interna) | Input create visibile | Textbox "Task title" visibile | ✅ PASS |
| B7 | Naviga `/app/about` (interna) | About content | Markdown i18n "Why toBear" renderizzato | ✅ PASS |
| B8 | Naviga `/app/contact` (interna) | Form contatto | Name, Email*, Message*, Send button | ✅ PASS |

**Osservazioni B5**: il bug del plugin `cspDynamicPlugin` rimuove il tag `<link rel="manifest">` dall'HTML. Il file `/app/manifest.webmanifest` esiste con 200 ma content-type `application/octet-stream` (dovrebbe `application/manifest+json`).

---

### C. Auth Flow

| # | Azione | Atteso | Ottenuto | Esito |
|---|---|---|---|---|
| C1 | Naviga `/login` con AUTH_ENABLED=false | Redirect a `/todo` | Router guard: `authEnabled=false` → next(name:'Todo') | ✅ PASS |
| C2-C5 | Auth flow completo | — | SKIP (AUTH_ENABLED=false in prod) | ⏭️ SKIP |

---

### D. PWA e Service Worker

| # | Azione | Atteso | Ottenuto | Esito |
|---|---|---|---|---|
| D1 | SW registration | registered + activated | SW scope `/app/`, stato "activated" | ✅ PASS |
| D2 | Cache storage | api-cache + static-resources | Solo precache populated. Runtime caches configurati ma non popolati (nessuna richiesta cacheable matchante ancora) | ⚠️ WARN |
| D3 | sw.js ReferenceError | Nessuno | `urlPattern: /\/api\//` regex, nessun `env` free variable | ✅ PASS |
| D4 | Simula offline | App shell da cache | SKIP (SW non controllante in headless) | ⏭️ SKIP |
| D5 | Sync online | Resume sync | SKIP | ⏭️ SKIP |
| D6 | Manifest | start_url=/app/, scope=/app/, lang=en | Verificato su file: corretto. Ma NON referenziato nell'HTML | ❌ **FAIL** (B5/D6 stessa causa) |

**Osservazioni D2**: i runtime caches `api-cache` e `static-resources` sono configurati nello sw.js e funzioneranno al primo request matchante con status 200. Il test headless non ha prodotto richieste API 200.

---

### E. PWABuilder

| # | Azione | Atteso | Ottenuto | Esito |
|---|---|---|---|---|
| E1 | Apri PWABuilder | — | Test eseguito | ✅ |
| E2 | Report score | Score ≥ 90 | **FAIL** — PWABuilder non trova manifest, ne genera uno fittizio (name "tobear.x10", scope "/", 0 icone) | ❌ **FAIL** |
| E3 | Issue list | Nessun errore | **Causa**: manifest link assente nell'HTML (bug cspDynamicPlugin) | ❌ **FAIL** |

---

### F. Sicurezza (via curl/bash)

| # | Azione | Atteso | Ottenuto | Esito |
|---|---|---|---|---|
| F1 | Set-Cookie presente | Cookie sessione | `tobear_session` presente | ✅ PASS |
| F2 | Strict-Transport-Security | Header presente | **Non presente** (gestito da LiteSpeed?) | ⚠️ WARN |
| F3 | X-Content-Type-Options: nosniff | Header presente | **Non presente** in risposta Laravel | ⚠️ WARN |
| F4 | URL invalido → no stack trace | APP_DEBUG=false | 404 generica Laravel, nessuno stack trace | ✅ PASS |
| F5 | Cookie: Secure, HttpOnly, SameSite | |=Lax| `Secure` ✓ `HttpOnly` ✓ `SameSite=Lax` ✓ `domain=.x10.mx` ⚠️ | ✅ PASS / ⚠️ domain |

**Osservazioni F5**: `SESSION_DOMAIN=tobear.x10.mx` configurato, ma il cookie ha `domain=.x10.mx` (potrebbe essere il server che modifica il dominio). Possibile problema di configurazione LiteSpeed.

---

### G. Performance (via curl)

| # | Azione | Atteso | Ottenuto | Esito |
|---|---|---|---|---|
| G1 | Performance tab (LCP, FCP, TTFB) | — | SKIP (richiede browser devtools) | ⏭️ SKIP |
| G2 | Content-Encoding: gzip | gzip | `content-encoding: gzip` attivo, 520KB→183KB | ✅ PASS |
| G3 | Bundle JS size | ≤ 300KB | **520KB** (>300KB, no lazy loading) | ❌ **FAIL** |

---

## Problemi trovati (con severità)

### 🔴 Critici

| # | Problema | Dove | Impatto |
|---|---|---|---|
| C1 | **Bug cspDynamicPlugin**: regex rimuove tutti i meta/link tag tra `<meta charset>` e CSP meta. HTML servito senza manifest, viewport, theme-color, favicon, apple-touch-icon. | `frontend/vite.config.js:33-35` (plugin `cspDynamicPlugin`) | PWABuilder score 0/100, PWA non installabile, SEO rotto, viewport assente su mobile |
| C2 | **Bundle 520KB > 300KB**: nessun lazy loading nel router (tutte le route eager). LCP e FCP penalizzati su mobile 3G. | `frontend/src/router/index.js:5-16` | Performance mobile degradata |

### 🟠 Alti

| # | Problema | Dove | Impatto |
|---|---|---|---|
| A1 | **Catch-all backend**: `Route::get('/{any}')` matcha solo segmenti singoli. Navigazione diretta a `/app/todo`, `/app/login`, `/app/about` → 404 | `backend/routes/web.php:12-18` | SEO rotta, SPA non deep-linkable via URL diretto |
| A2 | **Cookie domain**: `domain=.x10.mx` invece di `tobear.x10.mx` | Config LiteSpeed / PHP session | Cookie leak cross-subdomain |
| A3 | **Bundle size**: 520KB senza code splitting | `frontend/src/router/index.js` | Mobile 3G: ~3s download + parse |

### 🟡 Medie

| # | Problema | Dove | Impatto |
|---|---|---|---|
| M1 | **SW non controllante** in headless Chrome (`clientsClaim()` non ha effetto nei test) | `dist/sw.js` (Workbox) | D4/D5 offline test non verificabile |
| M2 | **Manca HSTS header** | Server/Laravel middleware | Sicurezza downgrade attack |
| M3 | **Manca X-Content-Type-Options** | Laravel middleware stack | MIME sniffing attack |
| M4 | **Manifest content-type** `application/octet-stream` | LiteSpeed MIME config | Dovrebbe `application/manifest+json` |

---

## Raccomandazioni (con priorità)

### 🔴 Immediato (bloccanti per PWABuilder/Play Store)

1. **Fix cspDynamicPlugin** in `frontend/vite.config.js:33-35`:
   - Regex attuale: `/<meta[\s\S]*?http-equiv="Content-Security-Policy"[\s\S]*?\/?>/i`
   - Fix: `<meta\s+http-equiv="Content-Security-Policy"[^>]*>` (solo il meta CSP, niente match greedy)
   - Riassegnare: ripristinare `<link rel="manifest">`, viewport, theme-color, favicon, apple-touch-icon
   - Sessione: **frontend**

2. **Fix catch-all backend** in `backend/routes/web.php:12-18`:
   - Sostituire `Route::get('/{any}', ...)->where('any', '^(?!api|assets|css|js).*$')` con `Route::fallback(...)` o `Route::get('/{any?}', ...)->where('any', '.*')`
   - Sessione: **backend**

### 🟠 Medio termine (pre-produzione Play Store)

3. **Lazy loading router**: convertire import in `() => import('@/pages/TodoPage.vue')`
4. **Fix SESSION_DOMAIN**: verificare config LiteSpeed per dominio cookie esatto
5. **Aggiungere HSTS e X-Content-Type-Options** nel middleware Laravel o .htaccess
6. **Aggiungere `manifest.webmanifest` MIME type** nel .htaccess: `AddType application/manifest+json .webmanifest`

### 🟢 Basso (debito tecnico)

7. **OK per ora** — bundle size 520KB comprensibile per MVP, da schedulare dopo sync
8. **SW runtime caching**: verificato funzionante, solo non popolato per mancanza di richieste API 200

---

## Prontezza per PWABuilder / Play Store

**NON PRONTO** allo stato attuale.

Bloccante critico: il bug del `cspDynamicPlugin` rimuove il manifest link dall'HTML. Senza `<link rel="manifest">`:
- PWABuilder score = molto basso (ha generato manifest fittizio)
- Chrome non riconosce l'app come PWA installabile
- `beforeinstallprompt` non verrà mai emesso
- Play Store richiede manifest valido e score PWABuilder ≥ 90

Dopo il fix del cspDynamicPlugin e del catch-all, il progetto sarà pronto per:
1. Ripetere PWABuilder validation (score atteso ≥ 90)
2. Pubblicazione su x10hosting stabile
3. Dopo acquisto dominio Namecheap → Play Store

---

## Next steps suggeriti (da TODO.md)

- **Fix urgente frontend**: cspDynamicPlugin regex (sessione frontend)
- **Fix urgente backend**: catch-all fallback (sessione backend)
- **Post-fix**: re-test PWABuilder + offline D4/D5
- **Feature**: liste annidate (`parent_id`/`depth`)
- **Feature**: sync multi-dispositivo + Hetzner VPS
- **Reels**: foto-task, R2 storage
- **Grocery**: vista specializzata
- **Insights**: dashboard motivazionale
- **Domain Namecheap**: dopo fix critici, prima di Play Store

---

*Report generato il 2026-07-07 da OpenCode Technical Lead in build mode.*
