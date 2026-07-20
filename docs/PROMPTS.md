# PROMPTS.md — Prompts iniciales por fase

> Prompts listos para copiar/pegar en **Claude Code** o **Antigravity** al inicio de cada fase o sesión.
> Cada prompt es autónomo: menciona los docs a leer, define el alcance, declara criterios de éxito y self-checks.
> El LLM debe **leer los docs referenciados antes de escribir código**. No lo asumas, dilo explícitamente en el prompt.
> Los docs viven en el repo `Yormun_Docs` (carpeta `Yormun/Yormun_Docs/`). Los prompts indican en qué **repo** se trabaja.

---

## Cómo usar este documento

1. Antes de cada fase, copia el prompt correspondiente y pégalo como primer mensaje en la sesión del IDE.
2. Espera a que el LLM lea los docs y proponga un plan.
3. Revisa el plan. Si hay dudas, pídelo por escrito.
4. Solo entonces autoriza a implementar.

Formato de los prompts:

- **`[CLAUDE CODE]`** — pegar en Claude Code.
- **`[ANTIGRAVITY]`** — pegar en Antigravity IDE.
- Cada uno referencia los docs específicos que debe leer y el repo donde se trabaja.

---

## Fase 1 — Base de infraestructura

### 1.1 [ANTIGRAVITY] — Provisionar OCI + K3s + Traefik (repo: Yormun_Infra)

```
Vas a arrancar la infraestructura base de YORMUNGANDER. Trabajas en el repo `Yormun_Infra`.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Yormun_Docs/docs/BLUEPRINT.md` sección 3 (Infraestructura y persistencia) y sección 5 (Dominios y exposición).
2. Lee `../Yormun_Docs/AGENTS.md` completo.
3. Lee `../Yormun_Docs/docs/WORKFLOW.md` sección 2 (mapa de ownership) para confirmar que este repo es tuyo.

TU TAREA:
Crear el setup inicial de infra que incluya:
- Manifests Kustomize para K3s: `k8s/base/` (Traefik, cert-manager, cloudflared, Postgres, Redis, Infisical).
- Overlays: `k8s/overlays/production/` con configs finales.
- Scripts en `scripts/bootstrap/` para: instalar K3s en la VM, configurar Cloudflare Tunnel, aplicar los manifests iniciales, y pre-pull de la imagen Deno en el nodo (no hay warm pool: los pods se crean bajo demanda y la imagen debe estar cacheada).
- README en la raíz del repo con el runbook paso a paso desde una VM Ubuntu limpia.

CRITERIOS DE ÉXITO:
- Todos los manifests pasan `kubectl apply --dry-run=server`.
- El README permite a un humano bootstrappear la VM desde cero en <2 h.
- Ningún secreto en los YAML: todos referencian Infisical.
- Postgres StatefulSet con volume 50 GB y extensión `vector` instalada en el init.
- cert-manager con ClusterIssuer Let's Encrypt configurado para challenge DNS-01, con wildcards para `*.yormun.com` y `*.yormungander.com`.
- Cada Deployment tiene readiness/liveness probes y resources declarados.
- Actualiza `../Yormun_Docs/STATUS.md` al empezar y al terminar.

RESTRICCIONES:
- Prohibido introducir dependencias no aprobadas.
- Prohibido usar `latest` como image tag: siempre versión pinneada.
- Prohibido `hostNetwork: true` en cualquier pod.

ANTES DE IMPLEMENTAR:
Propón el plan detallado (archivos que crearás, orden, decisiones no triviales). Espera mi aprobación.
```

### 1.2 [CLAUDE CODE] — Configurar backups y verificación (repo: Yormun_Infra)

```
Vas a implementar el sistema de backups y su verificación. Trabajas en el repo `Yormun_Infra` (excepción al ownership: los scripts de backup son críticos y los lidera Claude Code; coordina en STATUS.md).

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Yormun_Docs/docs/BLUEPRINT.md` secciones 3.3.1 (sqlite-vec) y 3.5 (Backups).
2. Lee `../Yormun_Docs/AGENTS.md` completo, énfasis en sección 5 (Seguridad) y sección 6 (Testing).
3. Lee `../Yormun_Docs/docs/WORKFLOW.md` para confirmar la coordinación.

TU TAREA:
Crear scripts y CronJobs para:
- `scripts/backup/dump-postgres.sh` — pg_dump custom format, cifrado con `age`, upload a R2 (bucket `yormun-backups`).
- `scripts/backup/dump-redis.sh` — SAVE + copia RDB, cifrado, upload.
- `scripts/backup/dump-memory.sh` — `sqlite3 memory.db ".backup"`, cifrado, upload (memoria extendida sqlite-vec).
- `scripts/backup/verify-restore.sh` — el día 1 de cada mes: levanta contenedor Postgres temporal, aplica el dump más reciente, valida integridad con checks SQL, borra el contenedor, envía resultado a Telegram (stub por ahora).
- CronJobs de Kubernetes en `k8s/base/backup/`.
- Tests que validen que:
  - El cifrado age funciona correctamente y no se puede descifrar sin la master key.
  - La rotación de retención (7 diarios, 4 semanales, 3 mensuales) funciona.
  - El restore test detecta corrupción intencional.

CRITERIOS DE ÉXITO:
- El backup completo se ejecuta en <10 min.
- El upload a R2 tolera fallos de red (retry con backoff).
- El test de restore corre en CI de este repo en cada PR que toca este código.
- Cobertura de tests >90% en los scripts críticos.

RESTRICCIONES:
- La master key de age nunca aparece en logs.
- Los dumps nunca se escriben a disco sin cifrar. Streaming pipe: `pg_dump | age | rclone`.

ANTES DE IMPLEMENTAR:
Propón el plan. Especifica cómo vas a probar el restore end-to-end sin datos reales.
```

---

## Fase 2 — Núcleo del orquestador

### 2.1 [ANTIGRAVITY] — Scaffolding de los repos (repos: Yormun_Core, Yormun_Executor, Yormun_Web, Yormun_CLI)

```
Vas a inicializar los repositorios de YORMUNGANDER. NO es un monorepo: cada repo es autónomo, con su propio CI, lockfile y `.nvmrc`.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Yormun_Docs/docs/BLUEPRINT.md` sección 4 completa (incluida 4.6, contratos entre repos).
2. Lee `../Yormun_Docs/AGENTS.md` sección 4 (Arquitectura de código).
3. Lee `../Yormun_Docs/docs/WORKFLOW.md` sección 2 (ownership por repo).

TU TAREA:
Para `Yormun_Core`:
- NestJS con Zod validation, pino logger, auth JWT stub, health endpoint.
- `@nestjs/swagger` configurado + script `generate:contract` que emite `contracts/openapi.json`.
- Estructura de módulos vacíos según AGENTS.md 4.1 (telegram, hitl, audit, budget, security, memory, model-provider, integrations, tools).
- Dockerfile, `.nvmrc` (Node 24), tsconfig strict según AGENTS.md 3, ESLint + Prettier + gitleaks pre-commit.
- GitHub Actions: lint, typecheck, test, generate:contract, build de imagen.

Para `Yormun_Executor`:
- NestJS separado con endpoint `/execute` stub (sin lógica de K8s todavía).
- Mismo tratamiento: OpenAPI export, Dockerfile, `.nvmrc`, CI propio.

Para `Yormun_Web` y `Yormun_CLI`:
- Repos con scaffold mínimo (Vite+React y Ink respectivamente), script `generate:api` (openapi-typescript contra el contrato de core), README, CI propio.

CRITERIOS DE ÉXITO:
- En cada repo por separado: `pnpm install && pnpm build && pnpm test` funciona desde cero.
- `pnpm dev` en core y executor levanta cada servicio con hot reload.
- CI verde en un PR draft de cada repo, de forma independiente.
- `pnpm generate:api` en web genera tipos válidos desde el contrato de core.
- Ningún `any` en el código generado.
- Ningún workspace de pnpm, ningún turbo.json, ningún paquete compartido.

RESTRICCIONES:
- Yormun_Core no importa `dockerode` ni `@kubernetes/client-node`.
- Nada de lógica de negocio real — solo boilerplate.
- Actualiza `../Yormun_Docs/STATUS.md`.

ANTES DE IMPLEMENTAR:
Propón la estructura de carpetas de cada repo y las dependencias iniciales. Espera aprobación.
```

### 2.2 [CLAUDE CODE] — HITL classifier + audit log (repo: Yormun_Core)

```
Vas a implementar dos piezas críticas de seguridad: el HITL classifier y el audit log con hash chain. Trabajas en el repo `Yormun_Core`.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Yormun_Docs/docs/BLUEPRINT.md` sección 9 (HITL) completa. Léela dos veces.
2. Lee `../Yormun_Docs/AGENTS.md` completo, especialmente secciones 5 y 6.
3. Lee `CLAUDE.md` sección 3 (áreas que lideras).
4. Verifica que Antigravity haya terminado la Fase 2.1 revisando `../Yormun_Docs/STATUS.md`.

TU TAREA:
Implementar en `src/hitl/`:
- `types.ts` — enum de niveles `auto | notify | confirm | dual-confirm`.
- `registry.ts` (en `src/tools/`) — declaración estática de tools con su `hitlLevel`. Empieza con 3 tools stub: `readEmails` (auto), `createCalendarEvent` (notify), `sendEmail` (confirm).
- `classifier.ts` — función `classifyToolCall(toolName, inputs): HITLDecision`.
- `classifier.spec.ts` — tests cubriendo 100% de la matriz. Incluye tests que verifiquen que el LLM NO puede cambiar el level en runtime.
- `dual-confirm.service.ts` — máquina de estados para las dos aprobaciones separadas por 30s.
- `timeout.service.ts` — expiración de aprobaciones pendientes.

Implementar en `src/audit/`:
- Schema Prisma o Drizzle para tabla `audit_log` (ver BLUEPRINT 9.5).
- `hash-chain.service.ts` — cálculo y verificación de hash.
- `audit.service.ts` — API para registrar acciones.
- Tests de mutación: intentar modificar una row histórica rompe la chain.
- CronJob de verificación diaria de la chain.

CRITERIOS DE ÉXITO:
- Cobertura HITL: 100%.
- Cobertura audit chain: 100% incluyendo casos de mutación.
- Tests corren en <5s cada uno.
- Los DTOs públicos anotados para OpenAPI (web/cli generarán sus tipos desde el contrato — no crees paquetes compartidos).
- ADR en `../Yormun_Docs/docs/adr/0001-hitl-four-levels.md` explicando la decisión de 4 niveles.

RESTRICCIONES:
- No implementes tools reales (Canvas, Google) todavía. Solo los stubs.
- No introduzcas librerías nuevas más allá de las ya aprobadas en `AGENTS.md`.
- Extended thinking activado para toda decisión no trivial.

USA CLAUDE OPUS 4.8 para esta tarea (es seguridad crítica).

ANTES DE IMPLEMENTAR:
Propón el diseño con firmas de funciones. Espera aprobación.
```

### 2.3 [CLAUDE CODE] — Executor con RBAC estricto (repo: Yormun_Executor)

```
Vas a implementar el Executor: microservicio separado que ejecuta código LLM-generado en pods aislados. Trabajas en el repo `Yormun_Executor` (es tuyo completo).

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Yormun_Docs/docs/BLUEPRINT.md` secciones 4.2, 4.3, 4.4 completas.
2. Lee `../Yormun_Docs/AGENTS.md` sección 5.3 (separación de privilegios).
3. Confirma en `../Yormun_Docs/STATUS.md` que la Fase 2.1 (scaffolding) está terminada.

TU TAREA:
Implementar en `src/`:
- `k8s.service.ts` — wrapper sobre `@kubernetes/client-node`. Métodos: `createPod`, `deletePod`, `waitForPod`, `getPodLogs`. Solo opera en namespace `agents-sandbox`.
- `rbac-validator.service.ts` — valida cada request contra whitelist de tools permitidas.
- `execute.controller.ts` — endpoint `POST /execute` con Zod validation. Recibe: `{tool, code, env, timeout, remote}`.
- `pod-lifecycle.service.ts` — ciclo de vida bajo demanda: crear pod al llegar la tarea, esperar readiness, ejecutar, recoger resultado, destruir. Sin warm pool. Verifica al startup que la imagen Deno está pre-pulled en el nodo y alerta si no.
- `modal.service.ts` — cliente para Modal API cuando `remote: true` (stub por ahora).
- Manifests Kubernetes (en el repo `Yormun_Infra`, `k8s/base/executor/`): ServiceAccount, Role, RoleBinding, NetworkPolicies. Coordina ese PR aparte.

CRITERIOS DE ÉXITO:
- Tests de RBAC: peticiones para tools fuera de whitelist son rechazadas con 403.
- Tests de aislamiento: un pod en `agents-sandbox` no puede alcanzar servicios en `yormun` (verificar con test de integration).
- Pod listo para ejecutar en <5 s con la imagen cacheada (medido desde la petición).
- Manifest de Role tiene solo permisos mínimos (create/get/list/delete pods en un namespace).
- `contracts/openapi.json` actualizado — core generará su cliente desde ahí.
- ADR: `../Yormun_Docs/docs/adr/0002-executor-separation.md`.

RESTRICCIONES:
- Nunca ejecutes código sin timeout.
- Nunca crees pods con `privileged: true` o `hostNetwork`.
- El SDK de K8s solo existe en este repo, jamás en Yormun_Core.

USA CLAUDE OPUS 4.8.

ANTES DE IMPLEMENTAR:
Propón el diseño de las interfaces HTTP y los manifests de RBAC. Espera aprobación.
```

### 2.4 [ANTIGRAVITY] — Telegram bot con grammY + ModelProvider (repo: Yormun_Core)

```
Vas a implementar dos módulos: el bot de Telegram y el abstraction layer de modelos LLM. Trabajas en el repo `Yormun_Core`.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Yormun_Docs/docs/BLUEPRINT.md` secciones 6, 8.2.
2. Lee `../Yormun_Docs/docs/MODEL_ROUTING.md` completo.
3. Lee `../Yormun_Docs/AGENTS.md` completo.

TU TAREA A:
Implementar `src/model-provider/`:
- `types.ts` — interface `ModelProvider` uniforme.
- `anthropic.provider.ts` — implementación para Claude.
- `google.provider.ts` — implementación para Gemini.
- `router.service.ts` — selector según `TaskProfile` desde `config/models.yaml`.
- `failover.service.ts` — retry + fallback.
- `config/models.yaml` — el archivo con los profiles (ver MODEL_ROUTING.md sección 2.1).
- Tests con mocks de ambas APIs.

TU TAREA B:
Implementar `src/telegram/`:
- Bot con grammY, webhook mode.
- Comandos: `/start`, `/status`, `/tasks`, `/approve <id>`, `/reject <id>`, `/budget`.
- Cards de HITL con botones inline.
- Auth: solo owner (chat_id whitelisted en Infisical).
- Integración con el HITL classifier de Claude (Fase 2.2).

CRITERIOS DE ÉXITO:
- Puedes enviar un mensaje al bot y recibir respuesta pasando por el ModelProvider.
- Un flujo de HITL confirm funciona end-to-end en Telegram.
- Tests de failover: si Anthropic 429, cambia a Gemini automáticamente.
- Todos los tokens en Infisical, no en env.

RESTRICCIONES:
- Ningún model ID hardcoded fuera de `models.yaml`.
- Ningún dato de mensajes se logea en plain (privacidad).
- Coordina con Claude Code sobre la interface del HITL classifier antes de consumirlo (mismo repo: revisa STATUS.md).

ANTES DE IMPLEMENTAR:
Propón el plan. Espera aprobación.
```

---

## Fase 3 — Primera integración end-to-end

### 3.1 [ANTIGRAVITY] — Integración Canvas LMS (repo: Yormun_Core)

```
Vas a integrar Canvas LMS usando la instancia de prueba del owner. Trabajas en el repo `Yormun_Core`.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Yormun_Docs/docs/BLUEPRINT.md` sección 7.1.
2. Lee `../Yormun_Docs/AGENTS.md` completo.
3. Confirma con el owner: qué URL de Canvas usar, qué token, qué curso de prueba.
4. Lee la documentación oficial de Canvas API vía MCP: https://canvas.instructure.com/doc/api/

TU TAREA:
Implementar `src/integrations/canvas/`:
- `client.ts` — cliente REST con auth por Personal Access Token (desde Infisical).
- `tools/list-assignments.tool.ts` — tool `auto` para listar tareas próximas.
- `tools/get-course-content.tool.ts` — tool `auto` para leer materiales.
- `tools/schedule-study-block.tool.ts` — tool `notify` para crear bloque en Calendar (delega a Google Calendar integration cuando exista).
- `shadowing.cron.ts` — cron 00:00 local que:
  1. Consulta Canvas por cambios en las últimas 24 h.
  2. Indexa cambios en pgvector.
  3. Genera resumen con Gemini 3.1 Pro (contexto largo por PDFs).
  4. Envía notificación Telegram con acciones sugeridas (HITL confirm).
- Tests con nock/mocks del API de Canvas.

CRITERIOS DE ÉXITO:
- Puedes correr manualmente el shadowing y recibir un resumen coherente en Telegram.
- Todo input externo (contenido de Canvas) pasa por el injection sanitizer.
- El audit log registra cada tool call con `external_inputs_summary`.
- Rate limiting: máximo 30 req/min contra Canvas API.

RESTRICCIONES:
- Solo lectura por ahora. Nada de submits ni respuestas a profesores.
- Prohibido enviar contenido a APIs externas sin envolverlo en `<untrusted_content>`.

USA GEMINI 3.1 PRO para el análisis (contexto largo por PDFs).

ANTES DE IMPLEMENTAR:
Propón el plan. Especifica cómo probarás sin depender de la API real de Canvas.
```

---

## Fase 4 — Guardrails avanzados

### 4.1 [CLAUDE CODE] — Budget guard + kill switch (repo: Yormun_Core)

```
Vas a implementar el sistema de budget: per-session, daily, y kill switch. Trabajas en el repo `Yormun_Core`.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Yormun_Docs/docs/BLUEPRINT.md` sección 9.6.
2. Lee `../Yormun_Docs/AGENTS.md` completo.
3. Lee `CLAUDE.md`.

TU TAREA:
Implementar `src/budget/`:
- `budget.service.ts` — tracking de tokens y dólares por session, per-day, global.
- `guard.middleware.ts` — intercepta cada llamada al ModelProvider y verifica presupuesto antes.
- `kill-switch.service.ts` — detección de runaway (>2× consumo/hora vs 24h previas).
- `unpause.controller.ts` — endpoint para reanudar tras kill switch (requiere confirm HITL).
- Métricas Prometheus: `tokens_consumed_total`, `budget_remaining_ratio`, `runaway_detected_total`.
- Alertas en Alertmanager → Telegram cuando budget al 80% y 100%.
- Tests: simular consumos crecientes y verificar cortes.

CRITERIOS DE ÉXITO:
- Test de runaway simulado: en 1 min se consumen X tokens; el kill switch dispara.
- Test de degradación: al 80% del daily, las tareas nuevas usan Haiku en vez de Opus.
- El kill switch bloquea todas las tools salvo `unpause` (que es confirm HITL).
- Cobertura >95%.

USA CLAUDE OPUS 4.8.

ANTES DE IMPLEMENTAR:
Propón el diseño incluyendo cómo el guard se integra con el ModelProvider sin acoplar demasiado.
```

### 4.2 [ANTIGRAVITY] — Integración Google Calendar + Gmail (repo: Yormun_Core)

```
Vas a integrar Google Calendar y Gmail con OAuth Testing. Trabajas en el repo `Yormun_Core`.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Yormun_Docs/docs/BLUEPRINT.md` sección 7.2.
2. Lee `../Yormun_Docs/AGENTS.md` completo.
3. Confirma con el owner: qué scopes, qué cuenta usar.

TU TAREA:
Implementar `src/integrations/google/`:
- `oauth.service.ts` — flujo OAuth Testing, refresh cada 7 días con cron que avisa 24h antes.
- `calendar/` — cliente + tools (list events, create event, update, delete).
- `gmail/` — cliente + tools (list messages, get thread, send).
- Clasificación HITL:
  - listar, leer: `auto`.
  - crear evento: `notify`.
  - responder correo: `confirm`.
  - enviar correo nuevo: `confirm`.
  - borrar evento pasado: `notify`. Borrar evento futuro: `confirm`.
- Tests con mocks del SDK googleapis.

CRITERIOS DE ÉXITO:
- OAuth funciona end-to-end (con refresh manual documentado en runbook).
- Puedes desde Telegram: "resume mis correos de hoy" y recibir un summary.
- Puedes desde Telegram: "responde a Juan que sí, con más info sobre X" y el bot pide confirm con la card antes de enviar.
- Ningún scope innecesario (mínimos posibles).

RESTRICCIONES:
- Contenido de correos envuelto en `<untrusted_content>`.
- Nunca logea contenido en plain.
- Rate limiting según los límites de Google APIs.

ANTES DE IMPLEMENTAR:
Propón el plan.
```

### 4.3 [CLAUDE CODE] — Memoria extendida con sqlite-vec (repo: Yormun_Core)

```
Vas a implementar la memoria extendida local del agente con sqlite-vec. Trabajas en el repo `Yormun_Core`.

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Yormun_Docs/docs/BLUEPRINT.md` secciones 3.3.1 y 6.4.
2. Lee `../Yormun_Docs/AGENTS.md` sección 5.1 (el contenido que entra a memoria también se sanitiza).
3. Lee `CLAUDE.md` sección 3 (src/memory es tuyo).

TU TAREA:
Implementar `src/memory/`:
- `memory.service.ts` — API pública: `recall(query, k, filters)`, `remember(entry)`, `consolidate(sessionId)`.
- `store.ts` — acceso a `memory.db` vía better-sqlite3 + extensión sqlite-vec. WAL mode. Un solo writer.
- Schema: tabla vec con embedding + columnas de metadata `tipo`, `fuente`, `fecha`, `modelo_embedding`.
- `consolidation.service.ts` — al cierre de cada sesión de agente, destila hechos/preferencias/lecciones (TaskProfile `memory_consolidation`) y los guarda.
- Recall al inicio de sesión: inyecta las k entradas más relevantes al contexto del agente.
- Tests: round-trip de embeddings, filtros de metadata, y que el contenido externo pasa por el sanitizer antes de persistir.

CRITERIOS DE ÉXITO:
- Una preferencia guardada en una sesión aparece en el recall de la siguiente (test E2E con mocks del LLM).
- Búsqueda KNN brute-force + filtros de metadata (NO dependas de los índices ANN de sqlite-vec: siguen en alpha).
- `memory.db` entra al backup nocturno (coordina con el script de Yormun_Infra, Fase 1.2).
- Solo este módulo importa better-sqlite3/sqlite-vec en todo el repo.
- Cobertura >85%.

RESTRICCIONES:
- Los pods efímeros jamás acceden a memory.db.
- Nada de memoria "automática" de todo: solo lo que la consolidación destila explícitamente.

USA CLAUDE OPUS 4.8 para el diseño del schema y el flujo de consolidación.

ANTES DE IMPLEMENTAR:
Propón el schema y las firmas de la API. Espera aprobación.
```

---

## Fase 5+ — Fases posteriores

Los prompts de las fases 5-7 (pods bajo demanda + Modal, interfaces avanzadas, hardening) se añaden a este documento cuando llegue el momento, siguiendo la misma estructura.

---

## Plantilla genérica

Cuando abras una nueva sesión no cubierta arriba, usa esta plantilla:

```
[NOMBRE DE LA TAREA] (repo: Yormun_XXX)

ANTES DE ESCRIBIR CÓDIGO:
1. Lee `../Yormun_Docs/docs/BLUEPRINT.md` sección [X].
2. Lee `../Yormun_Docs/AGENTS.md` completo.
3. Lee [otros docs relevantes].
4. Revisa `../Yormun_Docs/STATUS.md` y verifica que no hay colisión con el otro IDE (solo posible en Yormun_Core).

TU TAREA:
[Descripción específica con archivos exactos]

CRITERIOS DE ÉXITO:
- [Lista concreta y verificable]

RESTRICCIONES:
- [Lista de "no hacer"]

USA [MODELO] para esta tarea porque [razón].

ANTES DE IMPLEMENTAR:
Propón el plan detallado. Espera aprobación explícita antes de escribir código.
Actualiza `../Yormun_Docs/STATUS.md` al empezar y al terminar.
```

---

## Reglas para todos los prompts

1. **Siempre indicar el repo** donde se trabaja. Sin repo declarado, el LLM no escribe código.
2. **Siempre pedir el plan primero.** Nunca dejes que el LLM escriba código en el primer turno.
3. **Siempre citar los docs a leer.** No asumas que el LLM recuerda.
4. **Siempre especificar el modelo** si es un caso donde la elección importa.
5. **Siempre pedir actualización de `STATUS.md`** al inicio y al fin.
6. **Siempre incluir criterios de éxito verificables**, no aspiraciones vagas.
7. **Siempre listar restricciones explícitas**, no confíes en que el LLM las infiera de AGENTS.md.
