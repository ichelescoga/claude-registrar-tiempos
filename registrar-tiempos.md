# /registrar-tiempos — Registrar worklogs + completar campos en Jira

## Que hace
Skill unificada con DOS modos de operacion segun el input que reciba.

## Modos de operacion

### MODO A — Registrar tiempos (sin argumentos o con fecha)
```
/registrar-tiempos              → lee tracking de HOY
/registrar-tiempos 2026-06-15   → lee tracking de esa fecha
```
Lee el archivo de tracking del dia, detecta que ya se registro y que no, completa campos faltantes en cada subtarea, agrega comentarios y worklogs por segmento.

**Hace TODO:**
1. Audita y completa campos faltantes (CeCo, FixVersion, labels, fechas, estimacion)
2. Agrega descripcion estandar (solo tickets de trabajo sin descripcion)
3. Agrega comentario por segmento de tracking
4. Registra worklog por segmento con hora real
5. Ajusta estimacion si hace falta

### MODO B — Solo completar subtareas (con ticket keys)
```
/registrar-tiempos FN-5436 FN-5415 FN-4930
```
Solo audita y completa campos faltantes. NO registra worklogs, NO agrega comentarios ni descripcion.

**Hace:**
1. Lee cada ticket y su padre
2. Detecta campos faltantes (CeCo, FixVersion, labels, fechas, estimacion)
3. Muestra propuesta → aprueba → ejecuta

### Como detectar el modo
- Si `$ARGUMENTS` contiene ticket keys (patron `FN-\d+`) → **MODO B**
- Si `$ARGUMENTS` es una fecha (`YYYY-MM-DD`) → **MODO A** con esa fecha
- Si `$ARGUMENTS` esta vacio → **MODO A** con fecha de hoy

---

## MODO A — Input: Archivo de tracking del dia

La skill lee automaticamente el archivo de tracking del dia desde memoria:
```
/Users/TRIBAL/.claude/projects/-Users-TRIBAL-Shigoto-Tribal-GIT-fri-tribal-bitbucket2/memory/timetracking_YYYY_MM_DD.md
```

Si no existe, pedir al usuario que pegue la tabla.

### Estructura obligatoria del archivo de tracking

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
| ... | ... | ... | ... | ... |
| | | | **TOTAL — Xh** | |
```

**Campos por fila:**
- `#` — numero secuencial
- `Hora` — rango horario real (HH:MM-HH:MM)
- `Actividad` — descripcion de lo que se hizo
- `Ticket` — clave Jira + nombre (formato: `FN-XXXX — Nombre` o `FN-XXXX`)
- `Horas` — duracion del segmento (formato: `0.25h`, `1h`, `1.5h`)

**Filas que se IGNORAN:**
- Filas sin ticket (ticket = `—` o vacio) → no se registran en Jira
- Filas de TOTAL/PARCIAL

## Conexion Jira API

- **URL base**: `https://tribal-mnc.atlassian.net`
- **Auth**: Basic Auth
  - Email: `gichel@tribalworldwide.gt`
  - API Token: leer de `reference_jira_access.md` en memoria del proyecto
- **API version**: v3 (`/rest/api/3/`)
- **Timezone**: `-0600` (Guatemala)
- **Account ID Gustavo**: `712020:98d8ea13-017e-4678-8f8d-97504460d4d6`

## Deteccion de duplicados — NO registrar lo que ya existe

ANTES de registrar un worklog, verificar si ya existe uno para ese ticket en esa fecha/hora:

```
GET /rest/api/3/issue/{KEY}/worklog
```

**Comparar cada worklog existente contra el segmento a registrar:**
- Si ya existe un worklog con `started` en la misma fecha Y `timeSpentSeconds` similar (+-300s) → SALTAR ese segmento
- Mostrar `[YA REGISTRADO]` en la propuesta para esos segmentos

## PASO 0: Validacion de conexion (ejecutar SIEMPRE al inicio)

Antes de hacer cualquier cosa, validar que tenemos acceso a Jira:

```bash
curl -s -o /dev/null -w "%{http_code}" "https://tribal-mnc.atlassian.net/rest/api/3/myself" \
  -u "{EMAIL}:{API_TOKEN}" -H "Content-Type: application/json"
```

- **200** → OK, continuar
- **401** → Token invalido o expirado. Leer `reference_jira_access.md` de memoria y reintentar. Si sigue fallando, avisar al usuario: "Token de Jira invalido, verifica reference_jira_access.md"
- **403** → Sin permisos. Avisar al usuario.
- **Timeout/error red** → Avisar: "No se puede conectar a Jira, verificar conexion"

**NO continuar si la validacion falla.**

---

## Clasificacion de tickets

### Tickets RECURRENTES (solo worklog con comentario)
Detectar por el summary del ticket. Si contiene alguna de estas palabras:
- "Daily"
- "Sesiones de Alineacion"
- "Ingreso de tiempos"
- "Banktime"
- "Checkpoint"
- "Revision de Tickets"
- "Revision PRs"
- "Soportes y Resolucion"

**Para recurrentes:**
- Solo worklog con comentario descriptivo
- NO descripcion del ticket
- NO comentario del issue

### Tickets de TRABAJO (todo el protocolo)
Cualquier ticket que NO sea recurrente.

**Para trabajo:**
1. Descripcion estandar (si no tiene)
2. Comentario por segmento
3. Worklog por segmento
4. Verificar campos (CeCo, FixVersion, labels, fechas, estimacion)

## Flujo de ejecucion

### PASO 1: Leer archivo de tracking
- Detectar fecha del dia actual o usar $ARGUMENTS si se pasa una fecha
- Leer el archivo `timetracking_YYYY_MM_DD.md`
- Parsear la tabla: extraer filas validas (con ticket)
- Agrupar segmentos por ticket

### PASO 2: Por cada ticket unico, leer estado actual
```
GET /rest/api/3/issue/{KEY}?fields=summary,status,issuetype,parent,assignee,labels,customfield_10015,customfield_10140,duedate,timetracking,customfield_12912,fixVersions,description
```

### PASO 3: Leer worklogs existentes
```
GET /rest/api/3/issue/{KEY}/worklog
```
- Marcar segmentos que ya tienen worklog como `[YA REGISTRADO]`

### PASO 4: Leer ticket PADRE (para campos heredables)
```
GET /rest/api/3/issue/{PARENT_KEY}?fields=summary,customfield_12912,fixVersions
```

### PASO 5: Determinar acciones por ticket

Para cada ticket, construir lista de acciones:

**Campos faltantes (PUT unico):**

| Campo | Field ID | Falta si... | Fuente |
|-------|----------|-------------|--------|
| Assignee | `assignee.accountId` | null o no es Gustavo | `712020:98d8ea13-017e-4678-8f8d-97504460d4d6` |
| Labels | `labels` | array vacio | Detectar del CeCo del padre (ver tabla) |
| Start Date | `customfield_10015` | null | Fecha del primer segmento del dia |
| End Date | `customfield_10140` | null | Fecha del ultimo segmento del dia |
| Due Date | `duedate` | null | = End Date |
| Estimacion | `timetracking.originalEstimate` | null o menor que timeSpent total | Solo subir, nunca bajar |
| CeCo | `customfield_12912` | null y padre lo tiene | Heredar id del padre |
| Fix Version | `fixVersions` | vacio y padre lo tiene | Heredar del padre |

**Labels por CeCo del padre:**

| CeCo contiene | Labels |
|---------------|--------|
| `FRI` o `FRIMTE` | `["FRI", "MTE", "WEB"]` |
| `BIAVE` o `AVE` | `["AVE", "backend"]` |
| `ONF`, `OF`, `GTH` | No cambiar |

**Descripcion (solo tickets trabajo sin descripcion):**
Formato ADF:
```
🔹 Fecha: DD/MM/YYYY
🔹 Proyecto: [detectar de labels: FRI Banrural / AVE BI]
🔹 Subtarea asignada: [nombre ticket padre]
🔹 Estado actual: En curso
🔹 Tiempo invertido: [suma segmentos]h

1️⃣ Avances del dia
📌 [bullets con detalle de cada segmento]

🧪 Pruebas realizadas
- [si aplica]
```

**Comentarios (solo tickets trabajo, uno por segmento):**
```
POST /rest/api/3/issue/{KEY}/comment
Body ADF: "📅 DD/MM — HH:MM-HH:MM (Xh)\n[Detalle de lo que se hizo en ese segmento]"
```

**Worklogs (todos los tickets, uno por segmento):**
```
POST /rest/api/3/issue/{KEY}/worklog
{
  "timeSpentSeconds": [segundos],
  "started": "YYYY-MM-DDTHH:MM:00.000-0600",
  "comment": { ADF con descripcion del segmento }
}
```

### PASO 6: Mostrar propuesta COMPLETA

SIEMPRE mostrar todo antes de ejecutar:

```
## Propuesta de registro — DD/MM/YYYY

### Campos a completar
| Ticket | Campo | Actual | Nuevo |
|--------|-------|--------|-------|
| FN-5402 | CeCo | NULL | TWW-TRB-ONF... |

### Worklogs a registrar
| # | Hora | Ticket | Horas | Jira |
|---|------|--------|-------|--------|
| 2 | 08:15-08:45 | FN-5402 | 0.5h | POR REGISTRAR |
| 3 | 08:45-09:00 | FN-5401 | 0.25h | POR REGISTRAR |
| 4 | 09:00-09:45 | FN-5402 | 0.75h | YA REGISTRADO |

### Comentarios a agregar
| Ticket | Segmento | Contenido |
|--------|----------|-----------|
| FN-5402 | 08:15-08:45 | Reunion Dani: servicios AVE |

¿Apruebas? (si/no)
```

### PASO 7: Ejecutar y reportar

Ejecutar en este orden:
1. PUT campos faltantes
2. PUT descripcion (si aplica)
3. POST comentarios del issue (si aplica — uno por segmento, tickets trabajo)
4. POST worklogs

Reportar:
```
## Resultado

| Ticket | Accion | Jira |
|--------|--------|--------|
| FN-5402 | 3 worklogs + CeCo + FixVersion | OK |
| FN-5401 | 1 worklog | OK |

Total: X worklogs registrados, Y campos completados, Z comentarios agregados.
```

### PASO 8: Cierre de ticket (cuando el usuario lo indique)

Cuando el usuario diga que hay que cerrar un ticket de trabajo:

1. **Actualizar descripcion** con tiempo total real y estado final:
   - Cambiar `🔹 Estado actual: En curso` → `🔹 Estado actual: Completado`
   - Cambiar `🔹 Tiempo invertido: Xh` → al total real sumando todos los worklogs
   - Agregar resumen de avances si no estaba

2. **Ajustar estimacion** si el timeSpent real supera la estimacion original (regla: solo subir)

3. **Transicion a Done** (intentar id `31`, si falla intentar otras transiciones disponibles)

## Regla de Estimacion

- Estimacion existente **mayor** que real → **dejarla**
- Estimacion existente **menor** que real → **subirla** al real
- Sin estimacion → usar timeSpent real

## Transiciones disponibles

| Desde | Transicion ID | Hacia |
|-------|--------------|-------|
| Tareas por hacer | 2 | En curso |
| Cualquiera | 31 | Done |

**NOTA:** La transicion a "En curso" puede ser id `2` o `14` dependiendo del workflow del ticket. Si una falla, intentar la otra.

## Referencia rapida CeCos

| ID | Valor | Proyecto |
|----|-------|----------|
| 15118 | TWW-TRB-DES-FRI-FRIMTE#629A | FRI/MTE |
| 15115 | TWW-TRB-ONF-C4ONF-FENIXONF#31 | Operativo No Facturable |
| 15114 | TWW-TRB-OF-C4OF-FENIXOF#31 | Operativo Facturable |
| 15111 | TWW-TRB-DES-BI-BIAVE#117A | AVE (BI) |
| 16714 | TWW-TRB-GTH-C4GTH-C4GTH#31 | GTH (Banktime) |

## Referencia rapida Fix Versions

| ID | Nombre |
|----|--------|
| 14609 | Cluster 4 - Proyecto MTE - FRI 2026 |
| 14542 | Cluster 4 - Operativo no facturable 2026 |
| 14541 | Cluster 4 - operativo facturable 2026 |
| 14543 | Cluster 4 - GTH 2026 |
| 12823 | Cluster 4 - BI Asistente Ejecutivo Virtual 2025 |

## Anti-patrones

| No hacer | Hacer |
|----------|-------|
| Ejecutar sin propuesta | SIEMPRE propuesta + confirmacion |
| Registrar worklog duplicado | Verificar existentes primero |
| Bajar estimacion | Solo subir o dejar |
| Inventar CeCo/FixVersion | Solo heredar del padre |
| Poner descripcion en recurrentes | Solo worklogs con comentario |
| Registrar filas sin ticket | Saltar filas con ticket = — |
| Consolidar worklogs de segmentos | UN worklog por cada segmento |
| Modificar ticket padre | Solo leer, nunca escribir |
