# STATUS

## Última actualización: 2026-07-20 (America/Lima)

> **Modo solitario:** Claude Code trabaja solo. Antigravity no está corriendo todavía.
> Claude Code asumirá también las tareas etiquetadas `[ANTIGRAVITY]` en `docs/PROMPTS.md` cuando sean bloqueantes (1.1 y 2.1), dejándolo registrado aquí. Cuando Antigravity se incorpore, retoma el ownership normal de WORKFLOW.md sección 2.

## En progreso

### Claude Code

- **Repo:** Yormun_Infra
- **Rama:** `feature/claude/infra-base`
- **Descripción:** Fase 1.1 **terminada y en review**: PR #1 abierto (<https://github.com/JFrnck/Yormun_Infra/pull/1>). kube-linter 0 errores en local. Merge lo hace el owner (WORKFLOW 3.2). Siguiente tarea de Claude Code: Fase 1.2 (backups) sobre `main` una vez mergeado el PR.
- **Estado:** En pausa esperando review del owner.

### Antigravity

- No activo.

## Decisiones del owner (2026-07-19)

- **ORM: Drizzle** (recomendación de ANALISIS §5 aceptada). Formalizar con ADR en Fase 2.1.
- **Observabilidad Fase 1: mínima** — Prometheus + Loki + Grafana básicos; **Tempo y PgBouncer pospuestos** hasta que duelan (recomendación de ANALISIS §8 aceptada).

## Plan aprobado

0. **Paso previo** — commitear stubs en los 5 repos de app (+ `.gitignore` para `.DS_Store`), crear `Yormun_Docs/docs/adr/` y `docs/runbooks/`, commit de Yormun_Docs.
1. **Fase 1.1** [rol Antigravity, ejecuta Claude Code] — Yormun_Infra: manifests Kustomize + scripts bootstrap + README runbook.
2. **Fase 1.2** [Claude Code] — Yormun_Infra: scripts de backup + verify-restore + CronJobs + tests.
3. **Fase 2.1** [rol Antigravity, ejecuta Claude Code] — Scaffolding de Yormun_Core, Yormun_Executor, Yormun_Web, Yormun_CLI.
4. **Fase 2.2** [Claude Code] — Yormun_Core: HITL classifier + audit log (Opus 4.8).
5. **Fase 2.3** [Claude Code] — Yormun_Executor: RBAC estricto (Opus 4.8).

## Bloqueados / esperando

- Merge de PR #1 de Yormun_Infra (Fase 1.1) por el owner → desbloquea Fase 1.2.
- El scaffolding de Fase 2.1 está commiteado en ramas locales `feature/antigravity/scaffolding-base` de los 4 repos de app, pero **sin push ni PR** (verificado 2026-07-20). Falta push + PR + merge a `main` antes de que Fase 2.2/2.3 puedan arrancar sobre ese código (WORKFLOW 3.2 y 5).
- Ejecución real del bootstrap en la VM OCI la hace el owner (Claude Code solo escribe manifests/scripts).

## Recientemente completado (últimos 7 días)

- 2026-07-20: [Yormun_Infra] Fase 1.1 (infra base) terminada por Claude Code; PR #1 abierto, pendiente de merge.
- 2026-07-20: [Yormun_Core, Yormun_Executor, Yormun_Web, Yormun_CLI] Fase 2.1 completada (scaffolding base de apps y CI pipelines).
- 2026-07-19: [Yormun_Docs] Bundle de documentación inicial commiteado (`main`).
- 2026-07-19: [todos los repos] Repos creados con README inicial; stubs AGENTS.md/CLAUDE.md colocados (pendientes de commit).
