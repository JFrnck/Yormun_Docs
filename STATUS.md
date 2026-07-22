# STATUS

## Última actualización: 2026-07-21 (America/Lima)

> **Modo solitario:** Claude Code trabaja solo. Antigravity no está corriendo todavía.
> Claude Code asumirá también las tareas etiquetadas `[ANTIGRAVITY]` en `docs/PROMPTS.md` cuando sean bloqueantes (1.1 y 2.1), dejándolo registrado aquí. Cuando Antigravity se incorpore, retoma el ownership normal de WORKFLOW.md sección 2.

## En progreso

### Claude Code

- **Repo:** Yormun_Executor (próximo)
- **Descripción:** Fase 2.3 (RBAC estricto + ejecución aislada) — siguiente en el plan aprobado. No arrancada todavía.
- **Estado:** Sin rama abierta.

### Antigravity

- No activo.

## Pull Requests abiertos (todos esperando review/merge del owner)

| Repo | PR | Rama → base | Contenido |
| --- | --- | --- | --- |
| Yormun_Infra | [#1](https://github.com/JFrnck/Yormun_Infra/pull/1) | `feature/claude/infra-base` → `main` | Fase 1.1: manifests K3s, bootstrap, runbook |
| Yormun_Infra | [#2](https://github.com/JFrnck/Yormun_Infra/pull/2) | `feature/claude/infra-backups` → `feature/claude/infra-base` | Fase 1.2: backups + verify-restore (apilado sobre #1 — mergear #1 primero) |
| Yormun_Core | [#1](https://github.com/JFrnck/Yormun_Core/pull/1) | `feature/antigravity/scaffolding-base` → `main` | Fase 2.1: scaffolding NestJS |
| Yormun_Core | [#2](https://github.com/JFrnck/Yormun_Core/pull/2) | `feature/claude/hitl-audit` → `feature/antigravity/scaffolding-base` | Fase 2.2: HITL classifier + audit log (apilado sobre #1 — mergear #1 primero) |
| Yormun_Executor | [#1](https://github.com/JFrnck/Yormun_Executor/pull/1) | `feature/antigravity/scaffolding-base` → `main` | Fase 2.1: scaffolding NestJS |
| Yormun_Web | [#1](https://github.com/JFrnck/Yormun_Web/pull/1) | `feature/antigravity/scaffolding-base` → `main` | Fase 2.1: scaffolding Vite+React |
| Yormun_CLI | [#1](https://github.com/JFrnck/Yormun_CLI/pull/1) | `feature/antigravity/scaffolding-base` → `main` | Fase 2.1: scaffolding Ink |

Los PRs de Web y CLI todavía tienen nota de review pidiendo renombrar `"name": "temp-*"` en su `package.json`. En Core y Executor ya se corrigió.

## Decisiones del owner

- **2026-07-19 — ORM: Drizzle** (recomendación de ANALISIS §5 aceptada). Ya en uso en Yormun_Core desde la Fase 2.2 (PR #2).
- **2026-07-19 — Observabilidad Fase 1: mínima** — Prometheus + Loki + Grafana básicos; **Tempo y PgBouncer pospuestos** hasta que duelan (recomendación de ANALISIS §8 aceptada).
- **2026-07-21 — Testing: Vitest, no Jest.** El scaffolding de Fase 2.1 había quedado en Jest (default de `nest new`), mientras AGENTS.md 6.2 exige Vitest. El owner aprobó migrar. **Ya migrado en Core y Executor — si Antigravity vuelve a tocar estos repos, NO revertir a Jest.** Detalles técnicos: `vitest.config.ts` usa `unplugin-swc` (esbuild no emite metadata de decoradores que Nest necesita para DI) y `oxc: false` (Vitest 4 lo introdujo como transform por defecto, hay que apagarlo para que SWC sea el único). Imports explícitos de `vitest` en los specs (no `globals: true`).
- **2026-07-21 — BLUEPRINT 9.5 corregido (ADR 0002):** `audit_log` gana columna `request_id`; el estado "pendiente" vive en tabla mutable separada `pending_approvals`, nunca en el log insert-only. Ver `docs/adr/0002-audit-log-request-id.md`. **Ya implementado** en Yormun_Core PR #2.
- **2026-07-21 — ADR 0001 (4 niveles HITL) escrito y ya implementado** en Yormun_Core PR #2.

## Plan aprobado

0. **Paso previo** — ✅ hecho.
1. **Fase 1.1** [rol Antigravity, ejecuta Claude Code] — ✅ hecho, PR #1 Yormun_Infra.
2. **Fase 1.2** [Claude Code] — ✅ hecho, PR #2 Yormun_Infra.
3. **Fase 2.1** [rol Antigravity, ejecuta Claude Code] — ✅ hecho, PR #1 en los 4 repos de app.
4. **Fase 2.2** [Claude Code] — ✅ hecho, PR #2 Yormun_Core (HITL classifier + audit log). CI verde.
5. **Fase 2.3** [Claude Code] — Yormun_Executor: RBAC estricto (Opus 4.8). **Siguiente, no arrancada.**

## Bloqueados / esperando

- Review y merge de los 7 PRs abiertos por el owner. Orden de merge por repo:
  - Yormun_Infra: #1 → #2.
  - Yormun_Core: #1 (scaffolding) → #2 (HITL/audit).
  - Yormun_Executor, Yormun_Web, Yormun_CLI: PR único cada uno, independientes entre sí.
- Ejecución real del bootstrap en la VM OCI la hace el owner (Claude Code solo escribe manifests/scripts).
- Fase 2.3 se apilará sobre `feature/antigravity/scaffolding-base` de Yormun_Executor (no sobre `main`, que todavía no la tiene) — mismo patrón que Core.

## Fase 2.2 — resumen técnico (Yormun_Core PR #2)

Implementado según ADR 0001 (4 niveles HITL) y ADR 0002 (`request_id` + `pending_approvals`):

- `src/tools/registry.ts` — 3 tools stub con hitlLevel estático, `Object.freeze()`-ado.
- `src/hitl/` — `classifier.ts` (nunca decide por `inputs`, tool desconocida → `UnknownToolError`), `dual-confirm.service.ts` (estado persistido en Postgres, 30s reales entre aprobaciones), `timeout.service.ts` (nunca aprueba; descarta o escala/abandona).
- `src/audit/` — `hash-chain.ts` (puro), `audit.service.ts` (insert-only, advisory lock `pg_advisory_xact_lock` para serializar escrituras concurrentes entre réplicas durante rolling update), `chain-verification.service.ts` (cron diario, bloquea escrituras si detecta corrupción).
- **Infra nueva que no existía:** Drizzle + `pg` (primera vez que Core habla con Postgres), migraciones con `.down.sql` a mano, `src/config/` (Zod + `@nestjs/config`, fail-fast).
- **Tests:** 36 unitarios (100% matriz tool×nivel, 4 tests de mutación del hash chain) + 18 de integración con testcontainers (Postgres real: mutación vía SQL directo detectada, 10 escrituras concurrentes sin bifurcar la cadena, dual-confirm con 30s reales) + 1 e2e. Nuevo script `pnpm test:integration` (requiere Docker).
- **Gap de cobertura documentado:** `timeout.service.ts` en ~70% — los caminos 'escalate'/'abandon' no tienen integration test porque ninguna tool registrada usa `timeoutBehavior: 'escalate'` todavía (llega con Canvas, Fase 3). La lógica de decisión en sí tiene 100% de cobertura unitaria.

## CI de Yormun_Core / Yormun_Executor: bugs encontrados y corregidos (2026-07-21)

El owner reportó que el CI de ambos repos falló al pushear la Fase 2.1. Investigación encontró **cuatro problemas reales**, todos corregidos en los commits de la migración a Vitest:

1. **Causa raíz del fallo de CI:** pnpm 11 bloquea por defecto scripts de instalación no aprobados (`ERR_PNPM_IGNORED_BUILDS`). Fix: `pnpm-workspace.yaml` con `allowBuilds` (aprueba `unrs-resolver`/`@swc/core`, bloquea `@scarf/scarf` por ser telemetría de terceros).
2. **Grave, independiente del CI:** `node_modules/` (289M) y `dist/` estaban commiteados en git en ambos repos — el `.gitignore` heredado del Paso 0 solo ignoraba `.DS_Store`. Corregido: `.gitignore` estándar + `git rm --cached`.
3. **Bug funcional real en Executor:** `app.controller.ts` importaba `AppService` desde el archivo equivocado y tipaba la dependencia como `any` — rompía el DI de Nest en runtime.
4. **`@typescript-eslint/no-explicit-any` estaba en `'off'`** en el scaffolding — contradice AGENTS.md 3.2. Reactivado en `'error'`.

También corregido de paso: glob patterns rotos en `lint`/`format` (faltaba `/**/`), y `package.json` `name` de ambos repos (`temp-core`/`temp-executor` → `yormun-core`/`yormun-executor`).

**CI verificado en verde** en ambos repos, y también en el PR #2 de Yormun_Core (Fase 2.2).

## Recientemente completado (últimos 7 días)

- 2026-07-21: [Yormun_Core] Fase 2.2 completa (HITL classifier + audit log + Drizzle + config module). PR #2 abierto, CI verde. 36 tests unitarios + 18 de integración (testcontainers) + 1 e2e.
- 2026-07-21: [Yormun_Docs] ADR 0001 (4 niveles HITL) y ADR 0002 (`request_id`/`pending_approvals`) escritos; BLUEPRINT 9.5 corregido.
- 2026-07-21: [Yormun_Core, Yormun_Executor] Migración Jest→Vitest + fix de CI (pnpm allowBuilds) + fix de node_modules/dist trackeados + fix de bug de DI en Executor + reactivación de `no-explicit-any` + fix de globs de lint/format + rename de `package.json`. CI verde en ambos.
- 2026-07-21: PRs abiertos en los 4 repos de app para la Fase 2.1 (ramas ya existían en local desde el 2026-07-20, sin pushear).
- 2026-07-21: [Yormun_Infra] PR #2 abierto (Fase 1.2, backups) — la rama ya estaba lista desde el 2026-07-20.
- 2026-07-20: [Yormun_Infra] Fase 1.1 (infra base) terminada por Claude Code; PR #1 abierto.
- 2026-07-20: [Yormun_Core, Yormun_Executor, Yormun_Web, Yormun_CLI] Fase 2.1 completada por Antigravity (scaffolding base de apps y CI pipelines), commiteada en local.
- 2026-07-19: [Yormun_Docs] Bundle de documentación inicial commiteado (`main`).
- 2026-07-19: [todos los repos] Repos creados con README inicial; stubs AGENTS.md/CLAUDE.md colocados.
