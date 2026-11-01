# Setup opencode per toBear — istruzioni

## Cosa contiene questo archivio

Due alberi di file, uno per ogni repo:

```
backend/
  AGENTS.md
  .opencode/
    skills/
      laravel-conventions/SKILL.md
      sanctum-auth/SKILL.md
      task-resource-pattern/SKILL.md
    commands/
      handoff.md
      commit.md
      new-endpoint.md
  handoffs/          <- vuota, ci finiranno i file generati da /handoff

frontend/
  AGENTS.md
  .opencode/
    skills/
      vue-conventions/SKILL.md
      offline-sync-pattern/SKILL.md
      tailwind-styling/SKILL.md
    commands/
      handoff.md
      commit.md
      new-offline-feature.md
  handoffs/
```

## Come installarli

1. Copia il contenuto di `backend/` nella root di `tobear-laravel-backend` (commit incluso: `AGENTS.md` e `.opencode/` vanno versionati in git).
2. Copia il contenuto di `frontend/` nella root di `tobear-vuejs-frontend`.
3. Apri opencode in ciascuna repo separatamente (sono due progetti, due sessioni distinte).
4. Al primo avvio in ogni repo, opencode caricherà automaticamente `AGENTS.md` e renderà disponibili skill (`/skill-name` o auto-discovery) e comandi (`/handoff`, `/commit`, `/new-endpoint`, `/new-offline-feature`).

## Workflow consigliato (stesso pattern che usi per GEOMAPROOF)

All'inizio di ogni sessione, scrivi semplicemente:

> Leggi AGENTS.md, poi i due handoff più recenti.

opencode seguirà le istruzioni di bootstrap già scritte in AGENTS.md.

Alla fine di una sessione di lavoro:

> /handoff

Per committare:

> /commit

## Note

- Ho clonato e ispezionato entrambe le repo reali (struttura, route, model, store, pattern offline) per scrivere AGENTS.md e skill basate sul codice effettivo, non su assunzioni generiche di "Laravel standard" o "Vue standard".
- I file `.gitkeep` dentro `handoffs/` sono solo per far esistere la cartella vuota in git; il primo `/handoff` reale la popolerà.
- Se preferisci NON versionare gli handoff (contengono dettagli di lavoro interno), aggiungi `handoffs/*.md` al rispettivo `.gitignore` e tieni solo `.gitkeep`.
