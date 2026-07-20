# YORMUNGANDER (Yormun) — Documentación del proyecto

Este bundle contiene la documentación completa para arrancar y desarrollar YORMUNGANDER.

## Dónde va cada archivo

Todo el bundle vive en el repo **Yormun_Docs**. Colócalo así:

```
Yormun/                          # tu carpeta local (no es repo)
├── Yormun_Docs/
│   ├── README.md                # este archivo
│   ├── ANALISIS.md              # análisis del refinamiento 2026-07
│   ├── AGENTS.md                # directivas para ambos LLMs
│   ├── CLAUDE.md                # directivas específicas de Claude Code
│   ├── STATUS.md                # créalo vacío — coordinación entre IDEs
│   └── docs/
│       ├── BLUEPRINT.md         # plan maestro
│       ├── WORKFLOW.md          # coordinación entre IDEs
│       ├── MODEL_ROUTING.md     # qué modelo usar cuándo
│       ├── PROMPTS.md           # prompts por fase
│       ├── adr/                 # créalo vacío
│       └── runbooks/            # créalo vacío
├── Yormun_Core/                 # + stubs AGENTS.md y CLAUDE.md (ver STUBS.md)
├── Yormun_Executor/             # + stubs
├── Yormun_Web/                  # + stubs
├── Yormun_CLI/                  # + stubs
└── Yormun_Infra/                # + stubs
```

En cada repo de app pega los dos stubs correspondientes de `STUBS.md` (un `AGENTS.md` y un `CLAUDE.md` de 3 líneas que redirigen a Yormun_Docs). Sin ellos, un IDE abierto en un solo repo no carga las directivas.

Al terminar: commit + push de Yormun_Docs primero, luego de cada repo con sus stubs.

## Orden de lectura

### Para el owner (tú)

1. **ANALISIS.md** — el porqué de cada decisión del refinamiento (dominios, multi-repo, sqlite-vec, pros/contras).
2. **BLUEPRINT.md** — entiende la arquitectura completa.
3. **WORKFLOW.md** — entiende cómo se coordinan los dos IDEs.
4. **MODEL_ROUTING.md** — entiende qué modelo elegir cuándo.
5. **PROMPTS.md** — este es tu manual de operación.

### Para los LLMs (automático)

Claude Code y Antigravity leen `AGENTS.md` y (Claude Code) `CLAUDE.md` al abrir el workspace — el stub de cada repo los redirige a los canónicos de Yormun_Docs. En cada sesión, los prompts de `PROMPTS.md` les indican qué otros docs leer según la tarea.

## Los 6 repositorios

No hay monorepo. Cada repo tiene su CI, su lockfile y su `.nvmrc`:

| Repo | Contenido | Owner primario |
| --- | --- | --- |
| `Yormun_Docs` | Esta documentación + `STATUS.md` + ADRs + runbooks | Compartido |
| `Yormun_Core` | Orquestador NestJS (HITL, audit, budget, security, memory, telegram, integraciones) | Compartido |
| `Yormun_Executor` | Ejecución aislada (único que habla con K8s) | Claude Code |
| `Yormun_Web` | Dashboard React (deploy: Cloudflare Pages) | Antigravity |
| `Yormun_CLI` | CLI con Ink | Antigravity |
| `Yormun_Infra` | Manifests Kustomize, Flux, bootstrap, scripts de backup | Antigravity |

Nombres runtime en K8s (namespaces, services, imágenes, bucket) van en minúscula: `yormun-core`, `yormun-executor`, `yormun-backups`.

## Verificación del setup

1. Abre `Yormun/` en Claude Code y pregunta: "resume las reglas de oro del proyecto" — debería responder con la lista de `BLUEPRINT.md` sección 15.
2. Prueba similar en Antigravity con `AGENTS.md`.
3. Copia el primer prompt de `PROMPTS.md` (Fase 1.1) en Antigravity para arrancar la Fase 1.

## Dominios

- **yormun.com** → núcleo confiable: `api.`, `dash.`, `grafana.`
- **yormungander.com** → zona efímera de agentes: `app-{uuid}.`, previews.
- Regla: contenido generado por agentes jamás se sirve desde `yormun.com`. Ver `BLUEPRINT.md` 5.2.

## Filosofía en una línea

**Empezar por lo aburrido y crítico** (infra, backups, HITL, audit, budget) **antes que por lo divertido y visible** (dashboard con Monaco, applets, agentes autónomos).

El sistema es útil cuando puedes confiar en él. La confianza se construye con las capas de abajo, no con la interfaz de arriba.

## Cambios a la documentación

Todo cambio a estos docs se hace por PR a `Yormun_Docs` con revisión humana. La documentación es código y aplica el mismo estándar.

Cuando actualices un modelo en `MODEL_ROUTING.md`, actualiza también `config/models.yaml` en `Yormun_Core` en el mismo momento (PR de docs + PR espejo en core). Sin excepciones.
