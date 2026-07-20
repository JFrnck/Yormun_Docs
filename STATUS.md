# STATUS

## Última actualización: 2026-07-20 (America/Lima)

> **Modo solitario:** Claude Code trabaja solo. Antigravity no está corriendo todavía.
> Claude Code asumirá también las tareas etiquetadas `[ANTIGRAVITY]` en `docs/PROMPTS.md` cuando sean bloqueantes (1.1 y 2.1), dejándolo registrado aquí. Cuando Antigravity se incorpore, retoma el ownership normal de WORKFLOW.md sección 2.

## En progreso

### Claude Code

- **Repo:** Yormun_Infra
- **Rama:** `feature/claude/infra-base`
- **Descripción:** Fase 1.1 (rol Antigravity asumido por Claude Code en modo solitario, plan detallado aprobado por el owner 2026-07-20): manifests Kustomize base + overlays, scripts de bootstrap, README runbook. Versiones de imágenes verificadas contra Docker Hub/GitHub el 2026-07-20.
- **Archivos activos:**
  - `k8s/**`
  - `scripts/bootstrap/**`
  - `README.md`, `.github/workflows/**`, `.kube-linter.yaml`
- **Estado:** En desarrollo activo. No editar estos archivos.

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

- Aprobación del owner del plan detallado de Fase 1.1 (próximo mensaje de Claude Code).
- Ejecución real del bootstrap en la VM OCI la hace el owner (Claude Code solo escribe manifests/scripts).

## Recientemente completado (últimos 7 días)

- 2026-07-20: [Yormun_Core, Yormun_Executor, Yormun_Web, Yormun_CLI] Fase 2.1 completada (scaffolding base de apps y CI pipelines).
- 2026-07-19: [Yormun_Docs] Bundle de documentación inicial commiteado (`main`).
- 2026-07-19: [todos los repos] Repos creados con README inicial; stubs AGENTS.md/CLAUDE.md colocados (pendientes de commit).
