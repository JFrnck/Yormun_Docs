# STATUS

## Última actualización: 2026-07-21 (America/Lima)

> **Modo solitario:** Claude Code trabaja solo. Antigravity no está corriendo todavía.
> Claude Code asumirá también las tareas etiquetadas `[ANTIGRAVITY]` en `docs/PROMPTS.md` cuando sean bloqueantes (1.1 y 2.1), dejándolo registrado aquí. Cuando Antigravity se incorpore, retoma el ownership normal de WORKFLOW.md sección 2.

## En progreso

### Claude Code

- **Repo:** Yormun_Core (próximo)
- **Descripción:** Diseño de la Fase 2.2 (HITL classifier + audit log) propuesto al owner, esperando aprobación antes de implementar (PROMPTS.md 2.2 lo exige explícitamente por ser seguridad crítica).
- **Estado:** Sin rama abierta todavía — se crea `feature/claude/hitl-audit` (apilada sobre `feature/antigravity/scaffolding-base` de Yormun_Core, ver nota de merge order abajo) cuando el owner apruebe el diseño.

### Antigravity

- No activo.

## Pull Requests abiertos (todos esperando review/merge del owner)

| Repo | PR | Rama → base | Contenido |
| --- | --- | --- | --- |
| Yormun_Infra | [#1](https://github.com/JFrnck/Yormun_Infra/pull/1) | `feature/claude/infra-base` → `main` | Fase 1.1: manifests K3s, bootstrap, runbook |
| Yormun_Infra | [#2](https://github.com/JFrnck/Yormun_Infra/pull/2) | `feature/claude/infra-backups` → `feature/claude/infra-base` | Fase 1.2: backups + verify-restore (apilado sobre #1 — mergear #1 primero) |
| Yormun_Core | [#1](https://github.com/JFrnck/Yormun_Core/pull/1) | `feature/antigravity/scaffolding-base` → `main` | Fase 2.1: scaffolding NestJS |
| Yormun_Executor | [#1](https://github.com/JFrnck/Yormun_Executor/pull/1) | `feature/antigravity/scaffolding-base` → `main` | Fase 2.1: scaffolding NestJS |
| Yormun_Web | [#1](https://github.com/JFrnck/Yormun_Web/pull/1) | `feature/antigravity/scaffolding-base` → `main` | Fase 2.1: scaffolding Vite+React |
| Yormun_CLI | [#1](https://github.com/JFrnck/Yormun_CLI/pull/1) | `feature/antigravity/scaffolding-base` → `main` | Fase 2.1: scaffolding Ink |

Los PRs de Web y CLI todavía tienen nota de review pidiendo renombrar `"name": "temp-*"` en su `package.json`. En Core y Executor ya se corrigió (ver abajo).

## Decisiones del owner

- **2026-07-19 — ORM: Drizzle** (recomendación de ANALISIS §5 aceptada). Formalizar con ADR en Fase 2.1/2.2.
- **2026-07-19 — Observabilidad Fase 1: mínima** — Prometheus + Loki + Grafana básicos; **Tempo y PgBouncer pospuestos** hasta que duelan (recomendación de ANALISIS §8 aceptada).
- **2026-07-21 — Testing: Vitest, no Jest.** El scaffolding de Fase 2.1 había quedado en Jest (default de `nest new`), mientras AGENTS.md 6.2 exige Vitest. El owner aprobó migrar (no aceptar Jest ni actualizar AGENTS.md). **Ya migrado en Core y Executor — si Antigravity vuelve a tocar estos repos, NO revertir a Jest.** Detalles técnicos: `vitest.config.ts` usa `unplugin-swc` (esbuild no emite metadata de decoradores que Nest necesita para DI) y `oxc: false` (Vitest 4 lo introdujo como transform por defecto, hay que apagarlo para que SWC sea el único). Imports explícitos de `vitest` en los specs (no `globals: true`).

## Plan aprobado

0. **Paso previo** — ✅ hecho.
1. **Fase 1.1** [rol Antigravity, ejecuta Claude Code] — ✅ hecho, PR #1 Yormun_Infra.
2. **Fase 1.2** [Claude Code] — ✅ hecho, PR #2 Yormun_Infra.
3. **Fase 2.1** [rol Antigravity, ejecuta Claude Code] — ✅ hecho, PR #1 en los 4 repos de app.
4. **Fase 2.2** [Claude Code] — Yormun_Core: HITL classifier + audit log (Opus 4.8). **Diseño en curso, no implementado.**
5. **Fase 2.3** [Claude Code] — Yormun_Executor: RBAC estricto (Opus 4.8). Pendiente.

## Bloqueados / esperando

- Review y merge de los 6 PRs abiertos por el owner (orden: Infra #1 → Infra #2; los 4 de scaffolding son independientes entre sí).
- Ninguna rama de Fase 2.2/2.3 se ha creado aún — se apilarán sobre las ramas de scaffolding respectivas (no sobre `main`, que todavía no las tiene) para no bloquear el desarrollo mientras el owner revisa. Se documentará el orden de merge en cada PR, igual que con Infra #1/#2.
- Ejecución real del bootstrap en la VM OCI la hace el owner (Claude Code solo escribe manifests/scripts).
- **Diseño de Fase 2.2 (HITL + audit log): pendiente de aprobación del owner sobre un ajuste al BLUEPRINT.md 9.5** — el schema de `audit_log` no tiene forma de correlacionar la fila `pending` con su fila terminal (`approved`/`rejected`/`timeout`) sin violar la inmutabilidad del hash chain. Propuesta: añadir columna `request_id` + tabla mutable separada `pending_approvals`. Ver el mensaje de Claude Code del 2026-07-21 para el detalle completo. **No implementar HITL/audit hasta que el owner responda esto.**

## CI de Yormun_Core / Yormun_Executor: bugs encontrados y corregidos (2026-07-21)

El owner reportó que el CI de ambos repos falló al pushear la Fase 2.1. Investigación encontró **cuatro problemas reales**, todos corregidos en los commits de la migración a Vitest (ver Recientemente completado):

1. **Causa raíz del fallo de CI:** pnpm 11 bloquea por defecto scripts de instalación no aprobados (`ERR_PNPM_IGNORED_BUILDS`). Fix: `pnpm-workspace.yaml` con `allowBuilds` (aprueba `unrs-resolver`/`@swc/core`, bloquea `@scarf/scarf` por ser telemetría de terceros).
2. **Grave, independiente del CI:** `node_modules/` (289M) y `dist/` estaban commiteados en git en ambos repos — el `.gitignore` heredado del Paso 0 solo ignoraba `.DS_Store`. Corregido: `.gitignore` estándar + `git rm --cached`.
3. **Bug funcional real en Executor:** `app.controller.ts` importaba `AppService` desde el archivo equivocado y tipaba la dependencia como `any` — rompía el DI de Nest en runtime (por eso los tests fallaban, no solo por Jest/Vitest).
4. **`@typescript-eslint/no-explicit-any` estaba en `'off'`** en el scaffolding — contradice AGENTS.md 3.2. Reactivado en `'error'` (así se habría atrapado el bug #3 en lint, no en tests).

También corregido de paso: glob patterns rotos en `lint`/`format` (faltaba `/**/`), y `package.json` `name` de ambos repos (`temp-core`/`temp-executor` → `yormun-core`/`yormun-executor`).

**CI verificado en verde después de los fixes** en ambos repos (`gh pr checks`).

## Recientemente completado (últimos 7 días)

- 2026-07-21: [Yormun_Core, Yormun_Executor] Migración Jest→Vitest + fix de CI (pnpm allowBuilds) + fix de node_modules/dist trackeados + fix de bug de DI en Executor + reactivación de `no-explicit-any` + fix de globs de lint/format + rename de `package.json`. CI verde en ambos.
- 2026-07-21: PRs abiertos en los 4 repos de app para la Fase 2.1 (ramas ya existían en local desde el 2026-07-20, sin pushear).
- 2026-07-21: [Yormun_Infra] PR #2 abierto (Fase 1.2, backups) — la rama ya estaba lista desde el 2026-07-20.
- 2026-07-20: [Yormun_Infra] Fase 1.1 (infra base) terminada por Claude Code; PR #1 abierto.
- 2026-07-20: [Yormun_Core, Yormun_Executor, Yormun_Web, Yormun_CLI] Fase 2.1 completada por Antigravity (scaffolding base de apps y CI pipelines), commiteada en local.
- 2026-07-19: [Yormun_Docs] Bundle de documentación inicial commiteado (`main`).
- 2026-07-19: [todos los repos] Repos creados con README inicial; stubs AGENTS.md/CLAUDE.md colocados.
