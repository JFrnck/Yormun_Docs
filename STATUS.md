# STATUS

## Última actualización: 2026-07-23 (America/Lima)

> **Antigravity ya está activo** (ver abajo, Fase 3.1). Retoma ownership normal de `docs/WORKFLOW.md` sección 2 — Claude Code ya no asume tareas `[ANTIGRAVITY]` por defecto, salvo negociación puntual vía esta misma nota.

## En progreso

### Claude Code

- **Repo:** ninguno activo ahora mismo.
- **Descripción:** construyó los prerequisitos de Fase 3.1 ([PR #3](https://github.com/JFrnck/Yormun_Core/pull/3), mergeado) y revisó/verificó de forma independiente (no solo confiando en el self-report) el PR de Antigravity ([PR #4](https://github.com/JFrnck/Yormun_Core/pull/4), **mergeado**) antes de mergearlo: `pnpm lint`/`test`/`build`/`tsc --noEmit` corridos a mano sobre la rama, confirmó que los 3 puntos de la Ronda 2 quedaron bien resueltos y que `registry.ts` no fue tocado.
- **Próximo:** ninguno pendiente — **Fase 3.1 completa y mergeada a `main`.**

### Antigravity

- **Repo:** Yormun_Core
- **Rama:** `feature/antigravity/telegram-bot` (a crear)
- **Descripción:** Fase 2.4 — **Tarea B únicamente: bot de Telegram** (`src/telegram/`). La Tarea A de 2.4 (`src/model-provider/**`, ver `docs/PROMPTS.md` §2.4) ya quedó construida de rebote como prerequisito de Fase 3.1 (PR #3, mergeado) — no reconstruirla.
- **Estado:** 🔵 **Siguiente tarea, arrancando.** El owner eligió priorizar el bot de Telegram (Fase 2.4) antes que Fase 4.1/4.2/4.3 — sin él, el sistema HITL ya construido (4 niveles, dual-confirm, timeout) no tiene ningún canal real de notificación/aprobación. `timeout.service.ts` hoy solo loguea un warning ("notificación real llega en Fase 2.4") en vez de notificar de verdad.

## Fase 3.1 (completada, referencia)

`src/integrations/canvas/**` mergeado en PR #4. `CanvasClientService` (rate limit 30 req/min), `CanvasToolsService` (3 tools, ambas `auto` sanitizando con `wrapUntrustedContent`), `ShadowingService` (cron nocturno vía `ModelRouterService.complete('long_context', ...)`). Handoff pendiente: `canvasScheduleStudyBlock` lanza `CalendarNotImplementedError` (501) a la espera de Google Calendar (Fase 4.2).

## Feedback Ronda 2 para Antigravity — plan Fase 3.1 (Canvas), enviado 2026-07-23

Claude Code revisó el plan actualizado (ya consumiendo `wrapUntrustedContent`/`ModelRouterService` en vez de reimplementarlos) contra el código real de `src/model-provider/` y `src/security/` recién mergeado, y contra `AGENTS.md` §8.1. Ajustar antes de implementar:

1. **Bug bloqueante — la llamada a `ModelRouterService.complete()` no matchea la interfaz real.** El plan usa `complete('long_context', { prompt, systemPrompt })`. `ModelCompletionRequest` (`src/model-provider/model-provider.types.ts`) no tiene ningún campo `prompt`, y le faltan dos campos **requeridos** (`messages`, `maxOutputTokens`, `temperature`) — esto no compilaría. Forma correcta:
   ```ts
   await this.modelRouterService.complete('long_context', {
     systemPrompt: '...',
     messages: [{ role: 'user', content: wrappedContent }],
     maxOutputTokens: 8000, // = config/models.yaml → long_context.max_tokens_output
     temperature: 0.4,      // = config/models.yaml → long_context.temperature
   });
   ```
   El router no copia `maxOutputTokens`/`temperature` del profile automáticamente — es responsabilidad del caller pasarlos explícitos.
2. **El stub de `canvasScheduleStudyBlock` usa la excepción equivocada.** El plan dice seguir "el patrón `ModalService`", pero ese patrón (`Yormun_Executor/src/modal/errors.ts`) es `ModalNotImplementedError extends YormunError` — clase propia con `code`/`httpStatus`/`cause` — no `NotImplementedException` de `@nestjs/common`. `AGENTS.md` §8.1 exige que todo error tenga su clase custom extendiendo `YormunError`. Debe ser `CalendarNotImplementedError extends YormunError` (`code: 'CANVAS_CALENDAR_NOT_IMPLEMENTED'`, `httpStatus: 501`) en `src/integrations/canvas/errors.ts`.
3. **Falta sanitizar `canvasListAssignments`, no solo `canvasGetCourseContent`.** El plan envuelve con `wrapUntrustedContent` el resultado de `canvasGetCourseContent` pero no el de `canvasListAssignments` — y esa tool también es `hitlLevel: 'auto'` (su resultado vuelve directo al contexto del LLM como resultado de tool-call). Títulos/descripciones de tareas son contenido externo igual que el contenido de curso — `AGENTS.md` §5.1 exige envolver ambas.

Menor (no bloqueante): usar `.spec.ts` co-ubicado para los tests, no una carpeta `__tests__/` separada — es la convención del resto del repo (hitl, audit, model-provider).

Lo que el plan sí acierta en esta ronda: consumo correcto del sanitizer y el model-router sin reimplementarlos, reuso correcto de las 3 tools ya declaradas en `registry.ts`, `CANVAS_BASE_URL`/`CANVAS_API_TOKEN` requeridas con fail-fast, rate limit de 30 req/min, mocks HTTP en tests, y la referencia a "Fase 4.2" (`PROMPTS.md`) para Google Calendar es correcta.

## Feedback Ronda 1 para Antigravity — plan Fase 3.1 (Canvas), enviado 2026-07-23 (ya resuelto)

Claude Code revisó el plan propuesto (Canvas client + tools + shadowing cron) contra `AGENTS.md`, `docs/PROMPTS.md` §3.1, las golden rules de `docs/BLUEPRINT.md` §15 y el estado real del código en `Yormun_Core`. Ajustar antes de implementar:

1. **Sanitizador de injection incorrecto.** El plan propone un `canvas-sanitizer.ts` propio con tag genérico `<untrusted_content>`. `AGENTS.md` §5.1 (golden rule #6) exige una función **compartida** `wrapUntrustedContent(content, source, sessionNonce)` en `src/security/injection-sanitizer.ts`, con tag `<untrusted_content_{sessionNonce}>` (nonce por sesión + escape HTML) y `generateSessionNonce()`. Ese módulo **no existe todavía** (`src/security/` solo tiene `.gitkeep`). Además `Yormun_Docs/CLAUDE.md` §3 asigna `src/security/**` a Claude Code, no a Antigravity — coordinar antes de tocarlo, no reimplementar una versión propia y más débil dentro de `integrations/canvas/`.
2. **Falta la infraestructura de model routing.** `PROMPTS.md` §3.1 exige usar Gemini 3.1 Pro para el resumen del shadowing; golden rule #5 prohíbe modelos hardcoded — todo debe pasar por `model-provider/router.ts` + `config/models.yaml` (ver `docs/MODEL_ROUTING.md`). Verificado: `src/model-provider/` solo tiene `.gitkeep`, no existe `config/models.yaml`, no hay SDK de LLM instalado en `package.json`. El plan no menciona cómo se generaría el resumen — esto es un prerequisito real, no un detalle menor.
3. **Referencia rota en AGENTS.md §5.1:** cita "ver ADR 0002" para el diseño nonce del sanitizador, pero ADR 0002 es sobre `audit_log`/`request_id` (colisión de numeración heredada del bundle de docs original, antes de que existiera ningún ADR). Cuando se construya el sanitizador, escribir el ADR real (sería el 0004).
4. **Confirmar con el owner** URL de Canvas, token y curso de prueba antes de escribir código (paso explícito de `PROMPTS.md` §3.1, punto 3) — no asumir env vars sin esa conversación.
5. **`canvasScheduleStudyBlock` depende de Google Calendar, que no existe.** No hay módulo de Calendar en el repo. Ya existe un tool stub `createCalendarEvent` (Fase 2.2, sin implementación real) en `src/tools/registry.ts` — aclarar si `canvasScheduleStudyBlock` lo reusa o si son dos tools distintos, y en cualquier caso stubear la dependencia honestamente (patrón `ModalService` 501 de Yormun_Executor) en vez de una implementación parcial silenciosa.
6. **`CANVAS_BASE_URL`/`CANVAS_API_TOKEN` no deberían ser opcionales.** `AGENTS.md` línea 371 exige fail-fast si falta una variable requerida. Marcarlas opcionales permite que el módulo de Canvas quede a medias en silencio — siendo Fase 3 específicamente para habilitar Canvas, deberían ser requeridas.

Lo que el plan sí acierta: los 3 niveles HITL coinciden con blueprint/PROMPTS, el rate limit de 30 req/min es correcto, mockear el API de Canvas en tests es válido (`AGENTS.md` §6.3 permite mocks de APIs externas), y `external_inputs_summary` ya existe en el schema de `audit_log` — no hace falta migración ahí.

## Pull Requests — todos mergeados a `main`

| Repo | PR | Contenido | Estado |
| --- | --- | --- | --- |
| Yormun_Infra | [#1](https://github.com/JFrnck/Yormun_Infra/pull/1) | Fase 1.1: manifests K3s, bootstrap, runbook | ✅ mergeado |
| Yormun_Infra | ~~#2~~ → [#3](https://github.com/JFrnck/Yormun_Infra/pull/3) | Fase 1.2: backups + verify-restore | ✅ mergeado (ver nota abajo) |
| Yormun_Core | [#1](https://github.com/JFrnck/Yormun_Core/pull/1) | Fase 2.1: scaffolding NestJS | ✅ mergeado |
| Yormun_Core | [#2](https://github.com/JFrnck/Yormun_Core/pull/2) | Fase 2.2: HITL classifier + audit log | ✅ mergeado |
| Yormun_Executor | [#1](https://github.com/JFrnck/Yormun_Executor/pull/1) | Fase 2.1: scaffolding NestJS | ✅ mergeado |
| Yormun_Executor | [#2](https://github.com/JFrnck/Yormun_Executor/pull/2) | Fase 2.3: RBAC + ejecución aislada | ✅ mergeado |
| Yormun_Web | [#1](https://github.com/JFrnck/Yormun_Web/pull/1) | Fase 2.1: scaffolding Vite+React | ✅ mergeado |
| Yormun_CLI | [#1](https://github.com/JFrnck/Yormun_CLI/pull/1) | Fase 2.1: scaffolding Ink | ✅ mergeado |
| Yormun_Infra | [#4](https://github.com/JFrnck/Yormun_Infra/pull/4) | RBAC del ServiceAccount de yormun-executor (ADR 0003 punto 3, follow-up de Fase 2.3) | ✅ mergeado |
| Yormun_Core | [#3](https://github.com/JFrnck/Yormun_Core/pull/3) | Prerequisitos Fase 3.1: injection-sanitizer + model-provider (ADR 0004) | ✅ mergeado |
| Yormun_Core | [#4](https://github.com/JFrnck/Yormun_Core/pull/4) | Fase 3.1: integración Canvas LMS + Shadowing Académico | ✅ mergeado |

**Nota — Yormun_Infra #2 se reemplazó por #3:** al mergear #1 con `--delete-branch`, GitHub cerró automáticamente #2 porque su rama base (`feature/claude/infra-base`, la de #1) dejó de existir — efecto colateral no documentado de GitHub en PRs apilados, no una acción intencional. Un PR cerrado así no se puede reabrir ni re-apuntar vía API una vez cerrado. Recuperado abriendo #3 desde la misma rama head (`feature/claude/infra-backups`, intacta) directo contra `main`; contenido idéntico (26 archivos, 1128 inserciones), CI verde, mergeado normalmente.

**Lección para futuros PRs apilados** (aplicada ya en Core y Executor sin incidentes): mergear el PR padre **sin** `--delete-branch` → `gh pr edit <hijo> --base main` → verificar que el hijo quede limpio → **recién entonces** borrar la rama vieja del padre → mergear el hijo con `--delete-branch`.

Los `"name": "temp-*"` de `package.json` en Web y CLI ya no aplican como pendiente — no se volvió a tocar antes del merge; si sigue ahí, es follow-up menor, no bloqueante.

## Decisiones del owner

- **2026-07-19 — ORM: Drizzle** (recomendación de ANALISIS §5 aceptada). En uso en Yormun_Core desde la Fase 2.2 (PR #2).
- **2026-07-19 — Observabilidad Fase 1: mínima** — Prometheus + Loki + Grafana básicos; **Tempo y PgBouncer pospuestos** hasta que duelan.
- **2026-07-21 — Testing: Vitest, no Jest.** Migrado en Core y Executor. **Si Antigravity vuelve a tocar estos repos, NO revertir a Jest.**
- **2026-07-21 — BLUEPRINT 9.5 corregido (ADR 0002):** `audit_log` gana columna `request_id`; estado "pendiente" en tabla mutable separada `pending_approvals`. Implementado en Yormun_Core PR #2.
- **2026-07-21 — ADR 0001 (4 niveles HITL)** escrito e implementado en Yormun_Core PR #2.
- **2026-07-22 — Testing de Executor: K3s real vía testcontainers, no mocks del cliente de Kubernetes** (`@testcontainers/k3s`). El owner eligió explícitamente esta opción por sobre mockear — ver ADR 0003 punto 5. Costo aceptado: CI más lento (~35-75s típico; se observó flakiness de red anidada containerd-en-Docker en algunas corridas, mitigada con reintentos).
- **2026-07-22 — ADR 0003 (Executor: separación + hallazgo `deno eval`)** escrito e implementado en Yormun_Executor PR #2.
- **2026-07-23 — Canvas LMS: single-tenant confirmado, sin soporte multi-usuario.** El owner preguntó si Canvas soportaría "cambiar de usuario" (cuentas de terceros); se le presentaron 3 opciones (single-tenant / multi-cuenta propia / multi-tenant real) y confirmó mantener single-tenant, consistente con BLUEPRINT §1-2 ("plataforma personal", no multi-región/HA). Un solo Personal Access Token de Canvas (del owner) → Infisical → REST API, como ya especifica §7.1. **No implementar** `user_id` en `audit_log`/`pending_approvals`, aislamiento de memory por usuario, ni enrutamiento HITL multi-persona — si en el futuro se reconsidera, requiere un ADR nuevo porque cambia el modelo de seguridad completo del proyecto.
- **2026-07-23 — Prerequisitos de Fase 3.1 (sanitizer + model-provider): los construye Claude Code, no Antigravity.** El plan de Antigravity para Canvas proponía construir `src/security/injection-sanitizer.ts` y `src/model-provider/**` dentro de su propia rama — ambos son área exclusiva de Claude Code por `WORKFLOW.md` §2.2 (infraestructura compartida que futuras integraciones como Gmail/Telegram también necesitarán). El owner confirmó que Claude Code los construyera aparte primero; Antigravity los consume una vez mergeados. Ver PR #3 de Yormun_Core y ADR 0004.
- **2026-07-23 — Después de Fase 3.1, siguiente prioridad: terminar Fase 2.4 (bot de Telegram), no Fase 4.x.** La Tarea A de Fase 2.4 (`model-provider`) ya quedó hecha de rebote en PR #3. Queda solo la Tarea B (`src/telegram/`, grammY). El owner la priorizó sobre Budget guard (4.1)/Google Calendar (4.2)/Memoria (4.3) porque el sistema HITL ya construido no tiene todavía ningún canal real de notificación/aprobación.

## Plan aprobado

0. **Paso previo** — ✅ hecho.
1. **Fase 1.1** [rol Antigravity, ejecuta Claude Code] — ✅ hecho, PR #1 Yormun_Infra.
2. **Fase 1.2** [Claude Code] — ✅ hecho, PR #2 Yormun_Infra.
3. **Fase 2.1** [rol Antigravity, ejecuta Claude Code] — ✅ hecho, PR #1 en los 4 repos de app.
4. **Fase 2.2** [Claude Code] — ✅ hecho, PR #2 Yormun_Core (HITL classifier + audit log). CI verde.
5. **Fase 2.3** [Claude Code] — ✅ hecho, PR #2 Yormun_Executor (RBAC + ejecución aislada). CI verde, mergeado.

**Fin de la Fase 2 del roadmap.** Los 8 PRs de las Fases 1-2 están revisados y mergeados a `main` en los 6 repos. Lo que sigue (BLUEPRINT §14) es la Fase 3 (Canvas LMS + Shadowing Académico, lidera Antigravity) — no arrancada.

## Bloqueados / esperando

- Ejecución real del bootstrap en la VM OCI la hace el owner (Claude Code solo escribe manifests/scripts).
- Decisión del owner sobre si arrancar Fase 3 ahora (asumiendo Claude Code también el rol de Antigravity, como en Fases 1.1/2.1) o esperar a que Antigravity esté disponible.

## Fase 2.3 (follow-up) — RBAC de NetworkPolicy en Yormun_Infra (PR #4)

Cierra ADR 0003 punto 3. `k8s/base/executor/`: `ServiceAccount` `executor` en `yormun-executor`, `Role` `executor-agents-sandbox` en `agents-sandbox` (`create/get/list/delete` de Pods per BLUEPRINT 4.2 + `create/delete` de NetworkPolicies, la extensión que ADR 0003 identificó como necesaria), `RoleBinding` conectando ambos across namespaces. Sin acceso a Secrets/ConfigMaps/Deployments/otros namespaces. Verificado con `kubectl kustomize k8s/base` + CI verde (lint manifests, shellcheck, bats, GitGuardian).

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

- 2026-07-23: [Yormun_Core] [PR #4](https://github.com/JFrnck/Yormun_Core/pull/4) mergeado — Fase 3.1 completada por Antigravity (integración Canvas LMS, cliente REST con rate limit de 30 req/min, handlers sanitizados con `wrapUntrustedContent`, ShadowingService nocturno consumiendo `ModelRouterService.complete('long_context', ...)` con Gemini 3.1 Pro, `CalendarNotImplementedError` 501). 95 tests en verde. Claude Code verificó de forma independiente (lint/test/build/typecheck a mano, no solo el self-report) antes de mergear — sin hallazgos nuevos.
- 2026-07-23: [Yormun_Core] PR #3 mergeado — prerequisitos de Fase 3.1: `src/security/injection-sanitizer.ts` (ADR 0004) + `src/model-provider/**` (router, failover, providers Anthropic/Google, config/models.yaml) + declaración de las 3 tools de Canvas en `registry.ts`. 87 tests, cobertura 97% en los módulos nuevos. Fix incidental de CI (env vars faltantes). Antigravity desbloqueado para retomar `integrations/canvas/**`.
- 2026-07-23: [Yormun_Infra] PR #4 mergeado — RBAC de NetworkPolicy para el ServiceAccount del Executor (ADR 0003 punto 3), cerrando el último follow-up técnico de la Fase 2.
- 2026-07-23: Review y merge de los 8 PRs de las Fases 1-2 en los 6 repos. Incidente: Yormun_Infra #2 auto-cerrado por GitHub al borrar la rama base de un PR apilado; recuperado como #3. Procedimiento corregido aplicado sin incidentes en Core y Executor. Todos los repos en `main` limpio, 0 PRs abiertos.
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
