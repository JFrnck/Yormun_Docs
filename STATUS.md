# STATUS

## Última actualización: 2026-07-22 (America/Lima)

> **Modo solitario:** Claude Code trabaja solo. Antigravity no está corriendo todavía.
> Claude Code asumirá también las tareas etiquetadas `[ANTIGRAVITY]` en `docs/PROMPTS.md` cuando sean bloqueantes (1.1 y 2.1), dejándolo registrado aquí. Cuando Antigravity se incorpore, retoma el ownership normal de WORKFLOW.md sección 2.

## En progreso

### Claude Code

- **Repo:** ninguno activo ahora mismo.
- **Descripción:** Fase 2.3 (Executor, RBAC + ejecución aislada) completada — PR abierto, esperando confirmación final de CI en GitHub Actions.
- **Próximo:** pendiente de que el owner decida — Fase 3 (Canvas, lidera Antigravity) o el PR de RBAC de NetworkPolicy en Yormun_Infra que Fase 2.3 dejó como follow-up.

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
| Yormun_Executor | [#2](https://github.com/JFrnck/Yormun_Executor/pull/2) | `feature/claude/executor-rbac` → `feature/antigravity/scaffolding-base` | Fase 2.3: RBAC + ejecución aislada (apilado sobre #1 — mergear #1 primero) |
| Yormun_Web | [#1](https://github.com/JFrnck/Yormun_Web/pull/1) | `feature/antigravity/scaffolding-base` → `main` | Fase 2.1: scaffolding Vite+React |
| Yormun_CLI | [#1](https://github.com/JFrnck/Yormun_CLI/pull/1) | `feature/antigravity/scaffolding-base` → `main` | Fase 2.1: scaffolding Ink |

Los PRs de Web y CLI todavía tienen nota de review pidiendo renombrar `"name": "temp-*"` en su `package.json`. En Core y Executor ya se corrigió.

## Decisiones del owner

- **2026-07-19 — ORM: Drizzle** (recomendación de ANALISIS §5 aceptada). En uso en Yormun_Core desde la Fase 2.2 (PR #2).
- **2026-07-19 — Observabilidad Fase 1: mínima** — Prometheus + Loki + Grafana básicos; **Tempo y PgBouncer pospuestos** hasta que duelan.
- **2026-07-21 — Testing: Vitest, no Jest.** Migrado en Core y Executor. **Si Antigravity vuelve a tocar estos repos, NO revertir a Jest.**
- **2026-07-21 — BLUEPRINT 9.5 corregido (ADR 0002):** `audit_log` gana columna `request_id`; estado "pendiente" en tabla mutable separada `pending_approvals`. Implementado en Yormun_Core PR #2.
- **2026-07-21 — ADR 0001 (4 niveles HITL)** escrito e implementado en Yormun_Core PR #2.
- **2026-07-22 — Testing de Executor: K3s real vía testcontainers, no mocks del cliente de Kubernetes** (`@testcontainers/k3s`). El owner eligió explícitamente esta opción por sobre mockear — ver ADR 0003 punto 5. Costo aceptado: CI más lento (~35-75s típico; se observó flakiness de red anidada containerd-en-Docker en algunas corridas, mitigada con reintentos).
- **2026-07-22 — ADR 0003 (Executor: separación + hallazgo `deno eval`)** escrito e implementado en Yormun_Executor PR #2.

## Plan aprobado

0. **Paso previo** — ✅ hecho.
1. **Fase 1.1** [rol Antigravity, ejecuta Claude Code] — ✅ hecho, PR #1 Yormun_Infra.
2. **Fase 1.2** [Claude Code] — ✅ hecho, PR #2 Yormun_Infra.
3. **Fase 2.1** [rol Antigravity, ejecuta Claude Code] — ✅ hecho, PR #1 en los 4 repos de app.
4. **Fase 2.2** [Claude Code] — ✅ hecho, PR #2 Yormun_Core (HITL classifier + audit log). CI verde.
5. **Fase 2.3** [Claude Code] — ✅ hecho, PR #2 Yormun_Executor (RBAC + ejecución aislada). CI corriendo.

**Fin de la Fase 2 del roadmap.** Lo que sigue (BLUEPRINT §14) es la Fase 3 (Canvas LMS + Shadowing Académico, lidera Antigravity) — no arrancada. Antes de eso: revisar/mergear los 8 PRs abiertos.

## Bloqueados / esperando

- Review y merge de los 8 PRs abiertos por el owner. Orden de merge por repo:
  - Yormun_Infra: #1 → #2.
  - Yormun_Core: #1 (scaffolding) → #2 (HITL/audit).
  - Yormun_Executor: #1 (scaffolding) → #2 (RBAC/ejecución aislada).
  - Yormun_Web, Yormun_CLI: PR único cada uno, independiente.
- **PR pendiente de crear en Yormun_Infra** (Fase 2.3 lo dejó como follow-up, ver ADR 0003 punto 3): el ServiceAccount del Executor necesita permiso `create/delete` sobre `networkpolicies` en `agents-sandbox`, además de lo ya especificado para Pods en BLUEPRINT 4.2. Sin esto, el mecanismo de whitelist de egreso por tool no puede aplicarse cuando exista la primera tool con egreso real.
- Ejecución real del bootstrap en la VM OCI la hace el owner (Claude Code solo escribe manifests/scripts).

## Fase 2.3 — resumen técnico (Yormun_Executor PR #2)

Implementado según ADR 0003 (separación de privilegios + hallazgo de seguridad `deno eval`):

- `src/rbac/` — whitelist propia (`runCode`, sin egreso), `RbacValidatorService` como única puerta de entrada (403 si no está en whitelist).
- `src/k8s/k8s.service.ts` — único punto de contacto con `@kubernetes/client-node` en todo el proyecto; `namespace` fijo al construir, estructuralmente no puede tocar otro namespace.
- **Hallazgo de seguridad real:** verificado contra Deno 2.9.3 real que `deno eval` tiene *"implicit access to all permissions"* — ignora `--allow-net` completamente. Corregido a `deno run` + código como `data:` URL en base64 (sin ConfigMap, sin shell).
- `src/pod-lifecycle/` — ciclo de vida bajo demanda, sin warm pool, siempre destruye el pod.
- `src/modal/` — stub explícito (501) para `remote: true`, Fase 5.
- **Tests:** 30 unitarios + 5 e2e (capa HTTP completa sin K8s real) + 2 de integración con **K3s real** (`@testcontainers/k3s`): camino feliz de ejecución, y el test de aislamiento de red real que pide PROMPTS.md (un pod en `agents-sandbox` con permiso de red de Deno explícito igual no alcanza un servicio en `yormun`, bloqueado por la NetworkPolicy real).
- **Fix incidental:** `app.controller.ts` (heredado de Fase 2.1) tenía un endpoint stub `@Post('execute')` que colisionaba de ruta con el `ExecuteController` real — toda petición a `/execute` devolvía 201 del stub, saltándose RBAC/Zod por completo. Corregido.

## Fase 2.2 — resumen técnico (Yormun_Core PR #2)

Implementado según ADR 0001 (4 niveles HITL) y ADR 0002 (`request_id` + `pending_approvals`):

- `src/tools/registry.ts` — 3 tools stub con hitlLevel estático, `Object.freeze()`-ado.
- `src/hitl/` — `classifier.ts` (nunca decide por `inputs`, tool desconocida → `UnknownToolError`), `dual-confirm.service.ts` (estado persistido en Postgres, 30s reales entre aprobaciones), `timeout.service.ts` (nunca aprueba; descarta o escala/abandona).
- `src/audit/` — `hash-chain.ts` (puro), `audit.service.ts` (insert-only, advisory lock para serializar escrituras concurrentes entre réplicas durante rolling update), `chain-verification.service.ts` (cron diario, bloquea escrituras si detecta corrupción).
- **Infra nueva que no existía:** Drizzle + `pg` (primera vez que Core habla con Postgres), migraciones con `.down.sql` a mano, `src/config/` (Zod + `@nestjs/config`, fail-fast).
- **Tests:** 36 unitarios (100% matriz tool×nivel, 4 tests de mutación del hash chain) + 18 de integración con testcontainers (Postgres real) + 1 e2e.
- **Gap de cobertura documentado:** `timeout.service.ts` en ~70% — los caminos 'escalate'/'abandon' no tienen integration test porque ninguna tool registrada usa `timeoutBehavior: 'escalate'` todavía (llega con Canvas, Fase 3).

## CI de Yormun_Core / Yormun_Executor: bugs encontrados y corregidos (2026-07-21)

El owner reportó que el CI de ambos repos falló al pushear la Fase 2.1. Investigación encontró **cuatro problemas reales**, todos corregidos:

1. **Causa raíz del fallo de CI:** pnpm 11 bloquea por defecto scripts de instalación no aprobados (`ERR_PNPM_IGNORED_BUILDS`). Fix: `pnpm-workspace.yaml` con `allowBuilds`.
2. **Grave, independiente del CI:** `node_modules/` (289M) y `dist/` estaban commiteados en git en ambos repos. Corregido: `.gitignore` estándar + `git rm --cached`.
3. **Bug funcional real en Executor:** `app.controller.ts` importaba `AppService` desde el archivo equivocado y tipaba la dependencia como `any` — rompía el DI de Nest en runtime.
4. **`@typescript-eslint/no-explicit-any` estaba en `'off'`** en el scaffolding — contradice AGENTS.md 3.2. Reactivado en `'error'`.

También corregido de paso: glob patterns rotos en `lint`/`format`, y `package.json` `name` de ambos repos.

**CI verificado en verde** en Core PR #1/#2 y Executor PR #1; Executor PR #2 esperando confirmación final.

## Recientemente completado (últimos 7 días)

- 2026-07-22: [Yormun_Executor] Fase 2.3 completa (RBAC + ejecución aislada con K3s real). PR #2 abierto. 30 unitarios + 5 e2e + 2 de integración (K3s real vía testcontainers).
- 2026-07-22: [Yormun_Docs] ADR 0003 (Executor: separación + hallazgo `deno eval`) escrito.
- 2026-07-21: [Yormun_Core] Fase 2.2 completa (HITL classifier + audit log + Drizzle + config module). PR #2 abierto, CI verde.
- 2026-07-21: [Yormun_Docs] ADR 0001 (4 niveles HITL) y ADR 0002 (`request_id`/`pending_approvals`) escritos; BLUEPRINT 9.5 corregido.
- 2026-07-21: [Yormun_Core, Yormun_Executor] Migración Jest→Vitest + fix de CI + fix de node_modules/dist trackeados + fix de bug de DI en Executor + reactivación de `no-explicit-any` + fix de globs de lint/format + rename de `package.json`.
- 2026-07-21: PRs abiertos en los 4 repos de app para la Fase 2.1.
- 2026-07-21: [Yormun_Infra] PR #2 abierto (Fase 1.2, backups).
- 2026-07-20: [Yormun_Infra] Fase 1.1 (infra base) terminada; PR #1 abierto.
- 2026-07-20: [Yormun_Core, Yormun_Executor, Yormun_Web, Yormun_CLI] Fase 2.1 completada por Antigravity (scaffolding base de apps y CI pipelines), commiteada en local.
- 2026-07-19: [Yormun_Docs] Bundle de documentación inicial commiteado (`main`).
- 2026-07-19: [todos los repos] Repos creados con README inicial; stubs AGENTS.md/CLAUDE.md colocados.
