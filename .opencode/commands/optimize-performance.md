# Optimize performance (PROMPT 7)

Sei il Technical Lead del progetto Tobear. Implementa le ottimizzazioni emerse dal report Lighthouse reale (Chrome DevTools, mobile emulated 4x CPU) per portare il Performance score da 50 a ≥ 85. Apri questa sessione nella root del super-repo (i file da modificare stanno in frontend/).

## Contesto

- **Dominio**: https://tobear.x10.mx
- **Repo**: `frontend/` (submodule, Vue 3 + Vite + vite-plugin-pwa)
- **Lighthouse reale mobile**: Performance 50, LCP 4.2s, TBT 2.160ms, FCP 2.4s, CLS 0
- **Causa principale**: bundle JS 511KB (index.js 320KB + TodoPage.js 191KB) su CPU 4x = 2.5s script evaluation
- **Server**: x10hosting LiteSpeed con **modulo Brotli attivo** (PHP + server-level)
- **Compressione attuale**: solo gzip via `vite-plugin-compression` (114KB per il bundle)

## Bootstrap (OBBLIGATORIO)

1. Leggi `docs/superpowers/reports/LH-REPORT-2026-07-07.md` — analisi completa Lighthouse con opportunità
2. Leggi `docs/superpowers/reports/RE-VALIDATION-2026-07-07.md` — stato attuale post-fix
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

Spostarlo in fondo al `<body>` oppure caricarlo post-load via `requestIdleCallback` (vedi esempio in PROMPT-archivio). **Verifica**: registerSW.js non deve apparire nei render-blocking resources di Lighthouse.

#### 3. fetchpriority="high" sul logo LCP

Il LCP element è `img/icons/icon-128x128.png` (visualizzata a 56x56px). Aggiungere `fetchpriority="high"` all img in `PwaInstallBanner.vue` o `MobileNavFab.vue`. **Verifica**: LCP breakdown `resourceLoadDelay` deve scendere da 2.5s a < 1s.

#### 4. Brotli compression (server + build)

**4a. `.htaccess`** (entrambi `frontend/public/.htaccess` e `backend/public/.htaccess`): blocco `BROTLI_COMPRESS` per js/css/html/json/manifest (vedi esempio in PROMPT-archivio).

**4b. `vite-plugin-compression`** (`vite.config.js`): seconda istanza del plugin con `algorithm: brotliCompress, ext: .br`.

**4c. `.htaccess` per servire .br files** (se server non comprime on-the-fly): rewrite rules Accept-Encoding br/gzip (vedi esempio in PROMPT-archivio).

**Verifica**: `curl -sH "Accept-Encoding: br" -I https://tobear.x10.mx/app/assets/index-*.js` → `content-encoding: br`. Transfer size da 114KB a ~90KB.

#### 5. Inline CSS critico o preload (`index.html`)

CSS principale render-blocking: preload con onload trick + noscript fallback (vedi esempio in PROMPT-archivio). **Verifica**: CSS non deve apparire nei render-blocking resources.

### P2 — Accessibility + SEO (impatto medio)

#### 6. Fix color contrast (CSS / design system)

- **PWA Install Banner**: testo `#6b6860` su `#2e2c28` (ratio 2.5:1) → schiarire a `#9a958a` o simile.
- **Bottone Install**: `#ffffff` su `bg-tb-accent` `#e8823a` (2.73:1) → scurire accent a `#c4682a`.
- **Verifica**: Accessibility da 90 a ≥ 95.

#### 7. Fix struttura lista `<li>` senza `<ul>` (`TodoPage.vue`)

Aggiungere `role="list"` al container di vuedraggable. **Verifica**: audit `listitem` deve passare.

#### 8. Fix canonical URL (`index.html`)

`<link rel="canonical" href="%VITE_BASE_URL%" />` non sostituito da Vite → URL assoluto `https://tobear.x10.mx/app/`. **Verifica**: audit `canonical` deve passare.

### P3 — Bonus (se tempo permette)

#### 9. Preconnect per Matomo (`index.html`): `<link rel="preconnect" href="https://stats.tobear.x10.mx" />`
#### 10. Target esnext (`vite.config.js`): `build.target: esnext` (rimuove ~19KB polyfills).

## Workflow

1. Per ogni task, modifica il file, verifica con `npm run build` e `npm run lint`
2. Testa localmente con `npm run dev` se possibile
3. Commit con Conventional Commits EN
4. **NON fare git push** — lo fa l utente dopo verifica
5. Scrivi handoff in `frontend/docs/superpowers/handoffs/2026-07-07-performance-optimization.md` (o data corrente)

## Verifica post-build (locale)

- `ls -lh dist/assets/*.js` — index < 150KB, vendor-*.js esistenti
- `ls -lh dist/assets/*.br` — .br generati
- `grep canonical dist/index.html` — href assoluto
- `grep fetchpriority dist/index.html` — presente
- `grep -i brotli dist/.htaccess` — presente

## Verifica post-deploy (su x10hosting)

- Brotli attivo: `curl -sH "Accept-Encoding: br" -I .../index-*.js | grep content-encoding`
- Transfer size: `curl -sH "Accept-Encoding: br" -o /dev/null -w "%{size_download}" ...`
- Canonical assoluto: `curl -s https://tobear.x10.mx/ | grep canonical`

## Output richiesto

1. **Handoff** in `frontend/docs/superpowers/handoffs/` (data corrente, formato `2026-07-07-performance-optimization.md`)
2. **Riepilogo modifiche** con file toccati e chunk sizes before/after
3. **Stima impatto** su score Lighthouse (target: Performance 75-85, Accessibility 95+, SEO 100)
4. **Commit** (no push) con messaggio `perf: optimize bundle splitting, compression, and a11y` (dettaglio task in PROMPT-archivio)

## File rilevanti

```
frontend/vite.config.js                        # manualChunks + brotli + target esnext
frontend/index.html                            # canonical, registerSW defer, preconnect, fetchpriority
frontend/public/.htaccess                      # brotli config + serve .br/.gz
backend/public/.htaccess                       # brotli config
frontend/src/pages/TodoPage.vue                # list structure role="list"
frontend/src/components/PwaInstallBanner.vue   # color contrast + fetchpriority
frontend/src/components/MobileNavFab.vue       # fetchpriority + color contrast
frontend/src/assets/themes.css                 # bg-tb-accent contrast fix (se serve)
```
