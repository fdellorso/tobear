# Lighthouse check (PROMPT 8)

Sei il Technical Lead del progetto Tobear. Verifica l impatto reale delle ottimizzazioni performance implementate (deployate su x10hosting). Apri questa sessione nella root del super-repo.

## Contesto

- **Dominio**: https://tobear.x10.mx
- **Repo**: `frontend/` (submodule, Vue 3 + Vite + vite-plugin-pwa)
- **Baseline pre-fix (Lighthouse mobile reale, 4x CPU)**: Performance 50, LCP 4.2s, TBT 2.160ms, FCP 2.4s, CLS 0; Accessibility 90, Best Practices 92, SEO 92

## Bootstrap (OBBLIGATORIO)

1. Leggi `frontend/AGENTS.md` (convenzioni + regola no-amend).
2. Leggi l handoff della sessione di ottimizzazione (`frontend/docs/superpowers/handoffs/`, il più recente su performance).
3. Leggi `frontend/vite.config.js` (config build attuale).
4. Solo dopo, esegui le verifiche.

## Task

### 1. Lighthouse mobile reale (Chrome DevTools, 4x CPU slowdown)

Esegui Lighthouse su `https://tobear.x10.mx/app/todo` con emulazione mobile. Confronta con la baseline:

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

### 2. Verifica compressioni e MIME type

- `curl -sH "Accept-Encoding: br" -I .../app/assets/index-*.js` → atteso `content-type: application/javascript` + `content-encoding: br`
- `curl -sH "Accept-Encoding: br" -I .../app/assets/index-*.css` → atteso `text/css` + `br`
- Transfer size vendor: `curl -sH "Accept-Encoding: br" -o /dev/null -w "%{size_download}" .../app/assets/vendor-*.js` → atteso ~138KB
- **REGRESSION CHECK**: se `content-type` è `application/octet-stream`, il brotli on-the-fly non funziona — investigare alternativa.

### 3. Verifica chunk sizes via DevTools Network panel

| File | Baseline (gzip) | Target (brotli) |
|---|---|---|
| index.js | 114KB | **~13KB** |
| vendor.js | (in index, 320KB) | **~138KB** |
| TodoPage.js (lazy) | 66KB | **~4.5KB** |
| CSS | 9.3KB | **~8KB** |

### 4. Audit Lighthouse specifici (checklist)

- [ ] `render-blocking-resources`: CSS NON deve apparire
- [ ] `unused-javascript`: index.js < 50KB unused
- [ ] `legacy-javascript`: nessun polyfill Babel (target esnext)
- [ ] `uses-rel-preconnect`: Matomo preconnect presente
- [ ] `canonical`: URL assoluto https://tobear.x10.mx/app/
- [ ] `color-contrast`: nessuna violazione
- [ ] `listitem` (a11y): PASS (role="list" su vuedraggable)

### 5. Verifica PWA / Service Worker

- SW raggiungibile: `curl .../app/sw.js` → 200
- SPA fallback: `curl .../app/sconosciuta` → 200 (SPA shell, non 404)

### 6. Esegui E2E tests

```bash
cd frontend
npm run test:e2e:pwa
# Atteso: 3 passed, 1 skipped (test.fixme offline)
```

### 7. Documenta risultati

Scrivi `frontend/docs/superpowers/handoffs/YYYY-MM-DD-lighthouse-verification.md` con:
- Screenshot Lighthouse (mobile reale 4x CPU) — se hai Playwright MCP, cattura uno screenshot
- Tabella metriche before/after/target
- Audit pass/fail
- Eventuali regressioni o problemi residui
- Raccomandazioni per ulteriori ottimizzazioni (se score < 85)

### 8. Se score < 85 (opzionale, solo se sotto target)

Colli residui: `unused-javascript` > 20KB → ulteriori split; LCP > 2.5s → preload hero o split vendor; TBT > 600ms → split vendor chunk; third-party dominante → Matomo (già async).

## Output richiesto

1. **Handoff** in `frontend/docs/superpowers/handoffs/`
2. **Riepilogo score** in tabella before/after/target
3. **Eventuale commit** (no --amend, no push) per fix residui (se score < 85)

## Regole operative

- NON usare `git commit --amend`.
- NON eseguire `git push` senza richiesta esplicita.
