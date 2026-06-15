# /registrar-tiempos — Claude Code Skill

Skill para Claude Code que automatiza el registro de tiempos y la verificacion de subtareas en Jira.

## Que hace

Una sola skill con dos modos de operacion:

| Modo | Comando | Que hace |
|------|---------|----------|
| **A** | `/registrar-tiempos` | Lee el tracking del dia y registra worklogs + completa campos |
| **A** | `/registrar-tiempos 2026-06-15` | Idem pero para una fecha especifica |
| **B** | `/registrar-tiempos FN-5436 FN-5415` | Solo audita y completa campos faltantes |

### Modo A — Registrar tiempos
- Lee archivo de tracking (`timetracking_YYYY_MM_DD.md`) desde memoria
- Detecta worklogs ya registrados (no duplica)
- Clasifica tickets: recurrentes (solo worklog) vs trabajo (worklog + descripcion + comentarios)
- Completa campos faltantes heredando del padre (CeCo, Fix Version, labels)
- Muestra propuesta completa → usuario aprueba → ejecuta

### Modo B — Completar subtareas
- Lee cada ticket y su padre
- Detecta campos faltantes (CeCo, Fix Version, labels, fechas, estimacion)
- Muestra propuesta → usuario aprueba → ejecuta

## Instalacion

### Opcion 1: Global (todos los proyectos)
```bash
cp registrar-tiempos.md ~/.claude/commands/
```

### Opcion 2: Por proyecto
```bash
cp registrar-tiempos.md {tu-proyecto}/.claude/commands/
```

## Requisitos

- Claude Code CLI
- API Token de Jira (Atlassian) con permisos de lectura/escritura
- Archivo de tracking en memoria del proyecto (para Modo A)

## Configuracion

La skill necesita las siguientes credenciales configuradas en un archivo de referencia accesible desde memoria:

- **URL Jira**: `https://{tu-instancia}.atlassian.net`
- **Email**: tu email de Atlassian
- **API Token**: token generado desde https://id.atlassian.com/manage-profile/security/api-tokens
- **Account ID**: tu account ID de Jira

## Campos que verifica y completa

| Campo | Source |
|-------|--------|
| Assignee | Configurado en la skill |
| Labels | Detectados del CeCo del padre |
| Start Date | Del primer worklog o fecha del tracking |
| End Date | Del ultimo worklog o fecha del tracking |
| Due Date | = End Date si no tiene |
| Estimacion Original | Del timeSpent real (solo sube, nunca baja) |
| Centro de Costos (CeCo) | Heredado del ticket padre |
| Fix Version | Heredado del ticket padre |
| Descripcion | Formato estandar (solo tickets de trabajo) |

## Estructura del archivo de tracking

```markdown
---
name: Timetracking DD/MM/YYYY
description: Registro de horas del dia
type: project
---

# Timetracking — [Dia] DD/MM/YYYY

| # | Hora | Actividad | Ticket | Horas |
|---|------|-----------|--------|-------|
| 1 | 08:00-08:45 | Descripcion actividad | FN-XXXX — Nombre subtarea | 0.75h |
| 2 | 08:45-09:00 | Daily | FN-5401 — Daily Junio | 0.25h |
| | | | **TOTAL — Xh** | |
```

## Protocolo de registro

### Tickets recurrentes (Daily, Sesiones, Ingreso tiempos, etc.)
- Solo worklog con comentario descriptivo
- Sin descripcion del ticket, sin comentarios del issue

### Tickets de trabajo
1. Descripcion estandar si no tiene
2. Comentario por cada segmento de tracking
3. Worklog por cada segmento con hora real
4. Campos faltantes completados

## Licencia

MIT
