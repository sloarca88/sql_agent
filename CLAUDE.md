# CLAUDE.md — Instrucciones para Claude Code
# Repositorio: sql-agents

> Este archivo define cómo Claude Code debe leer, modificar y crear
> archivos en este repositorio. Léelo completo antes de hacer cualquier cambio.

---

## 1. Descripción del Proyecto

Repositorio de **pipelines Text-to-SQL** desplegados como agentes en **OpenWebUI**.
Cada pipeline recibe una pregunta en lenguaje natural, genera una consulta SQL
usando OpenAI (Responses API), la valida, la ejecuta contra una base de datos,
y devuelve el resultado formateado como tabla Markdown — sin reenviar datos al LLM.

**Plataforma de despliegue:** OpenWebUI (pipeline server)
**Autor:** Sergio Loarca
**Lenguaje:** Python 3.10+

---

## 2. Stack Tecnológico

| Capa | Tecnología |
|---|---|
| Lenguaje | Python 3.10+ |
| Framework de pipelines | OpenWebUI Pipeline API |
| LLM | OpenAI GPT (Responses API — `client.responses.create`) |
| BD SQL Server | pyodbc + ODBC Driver 17 for SQL Server |
| BD PostgreSQL | psycopg2-binary (driver nativo, sin ODBC) |
| Validación de modelos | Pydantic v2 (`BaseModel`, `SecretStr`) |
| Variables de entorno | python-dotenv (dev) / variables de contenedor (prod) |
| Infraestructura (futuro) | Docker / docker-compose |

**Dependencias por tipo de agente:**
```bash
# Agentes SQL Server
pip install pyodbc openai pydantic python-dotenv

# Agentes PostgreSQL
pip install psycopg2-binary openai pydantic python-dotenv
```

---

## 3. Estructura del Repositorio

```
sql-agents/
├── agents/                         ← Un archivo .py por agente (monolítico)
│   ├── agent_sql_boletas_worki.py  ← Agente: boletas de pago (SQL Server)
│   ├── agent_sql_presupuesto.py    ← Agente: presupuesto (PostgreSQL)
│   └── agent_sql_<nombre>.py       ← Futuros agentes (mismo patrón)
│
├── env/                            ← Plantillas de variables de entorno
│   ├── .env.boletas_worki.example  ← Variables requeridas para boletas
│   ├── .env.presupuesto.example    ← Variables requeridas para presupuesto
│   └── .env.<nombre>.example       ← Una plantilla por agente
│
├── docs/                           ← Documentación técnica
│   ├── arquitectura.md             ← Descripción del patrón común de pipelines
│   ├── vistas/                     ← Documentación de vistas SQL por agente
│   │   ├── boletas_worki.md        ← Schema, columnas y reglas de negocio
│   │   └── presupuesto.md          ← Schema, vistas autorizadas y reglas
│   └── observabilidad.md           ← Descripción de la BD de observabilidad
│
├── .github/
│   └── PULL_REQUEST_TEMPLATE.md
│
├── .gitignore
├── CLAUDE.md                       ← Este archivo
└── README.md
```

> **Regla de estructura:** cada agente nuevo sigue exactamente este patrón.
> No crear subcarpetas dentro de `agents/` — cada pipeline es un archivo único.

---

## 4. Patrón Arquitectónico Común

Todos los agentes comparten la misma estructura interna. Claude debe
respetarla al modificar o crear pipelines:

```
Pipeline
├── Valves (Pydantic BaseModel)     ← Credenciales y configuración desde env vars
├── __init__()                      ← Inicializa self.name, self.conn, self.valves
├── on_startup()  [async]           ← Establece conexión a BD
├── on_shutdown() [async]           ← Cierra conexión a BD
│
├── _init_db_connection()           ← Conecta a la BD específica del agente
├── _build_schema()                 ← Construye el schema para inyectar en el prompt
├── _validate_sql(sql)              ← Whitelist de seguridad — NUNCA omitir
├── _generar_sql_openai(question)   ← Llama a OpenAI Responses API
├── _extract_sql(text)              ← Extrae el bloque SQL de la respuesta del LLM
├── _run_sql_query(sql)             ← Ejecuta el SQL en la BD
├── _process_sql_output(sql, rows)  ← Formatea resultado como tabla Markdown
├── _format_markdown_table(rows)    ← Convierte lista de dicts a tabla Markdown
├── _json_serial(obj)               ← Serializa datetime y Decimal
├── pipe(...)                       ← Punto de entrada requerido por OpenWebUI
└── _log(message)                   ← Log interno — NUNCA exponer al usuario
```

**Flujo dentro de `pipe()`:**
```
pregunta usuario
      ↓
_generar_sql_openai()    → LLM genera SQL
      ↓
_extract_sql()           → limpia la respuesta
      ↓
_validate_sql()          → whitelist de seguridad ← OBLIGATORIO, nunca saltar
      ↓
_run_sql_query()         → ejecuta en BD
      ↓
_process_sql_output()    → formatea Markdown
      ↓
respuesta al usuario     ← NUNCA contiene detalles técnicos de BD
```

---

## 5. Diferencias entre Tipos de Agente

Al modificar un agente, identificar primero su tipo:

### Agentes SQL Server (ej: boletas_worki)
```python
# Driver
import pyodbc

# Conexión
conn_str = (
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={host},{port};DATABASE={db};"
    f"UID={user};PWD={pwd};TrustServerCertificate=yes;"
)
conn = pyodbc.connect(conn_str, autocommit=True)

# Dialecto SQL — usar exclusivamente:
SELECT TOP 500 ...          # límite de filas (NO usar LIMIT)
YEAR(columna) = 2024        # filtro de año
MONTH(columna) = 3          # filtro de mes
column LIKE '%texto%'       # búsqueda (NO usar ILIKE)

# Schema
SCHEMA_COLUMNS = "..."      # hardcodeado en el archivo como constante global
```

### Agentes PostgreSQL (ej: presupuesto)
```python
# Driver
import psycopg2
import psycopg2.extras

# Conexión
conn = psycopg2.connect(
    host=host, port=port, user=user, password=pwd,
    dbname=db, connect_timeout=10,
    options="-c statement_timeout=30000"
)
conn.autocommit = True

# Dialecto SQL — usar exclusivamente:
SELECT ... LIMIT 500        # límite de filas (NO usar TOP)
EXTRACT(YEAR FROM col)      # filtro de año
col ILIKE '%texto%'         # búsqueda case-insensitive (NO usar LIKE con LOWER)
value::numeric              # casting de tipos

# Schema
_build_schema()             # dinámico desde pg_description en cada petición
```

---

## 6. Reglas de Seguridad — CRÍTICAS

Estas reglas son invariantes. Claude **nunca** debe modificarlas sin
confirmación explícita y justificación técnica documentada:

### 6.1 Validación SQL por whitelist
- Cada agente tiene una lista `SQL_FORBIDDEN_KEYWORDS` y la constante `SQL_REQUIRED_START = "SELECT"`
- `_validate_sql()` se llama **siempre** antes de ejecutar cualquier SQL generado
- Nunca mover, saltarse ni comentar esta validación
- Si se agregan keywords nuevos a la whitelist, agregarlos con comentario que explique el riesgo

### 6.2 Credenciales
- **Nunca hardcodear** credenciales, API keys ni tokens en el código
- Todas las credenciales vienen de `os.getenv("NOMBRE_VAR", "")` dentro de `Valves`
- Los valores de `SecretStr` se acceden con `.get_secret_value()` — nunca imprimir ni loggear el valor
- Cada agente tiene su prefijo de variable de entorno (ej: `BOLETAS_`, `PRESUPUESTO_`)

### 6.3 Mensajes al usuario
- Los errores técnicos de BD, API y sistema **nunca** llegan al usuario final
- `_log()` es para mensajes internos; el `return` de `pipe()` es para el usuario
- Mensajes al usuario: genéricos, en español, sin stack traces ni nombres de tablas/columnas internas

### 6.4 Datos sin reenviar al LLM
- Los resultados de la BD **nunca** se reenvían al LLM para reformateo
- `_process_sql_output()` formatea directamente sin llamada adicional a OpenAI
- Esto es intencional por privacidad de datos

---

## 7. Convenciones de Código

### Nombrado
- **Clases:** PascalCase → `Pipeline`, `Valves`, `AgentEvent`
- **Métodos privados:** snake_case con prefijo `_` → `_validate_sql`, `_run_sql_query`
- **Constantes globales:** UPPER_SNAKE_CASE → `SQL_FORBIDDEN_KEYWORDS`, `SCHEMA_COLUMNS`
- **Variables locales:** snake_case → `sql_raw`, `conn_str`, `row_count`
- **Variables de entorno:** `PREFIJO_NOMBRE_VAR` → `BOLETAS_DB_HOST`, `PRESUPUESTO_DB_VIEW`

### Estilo Python
- Comillas dobles `"` para strings (consistente con archivos existentes)
- Indentación: 4 espacios
- Alineación vertical en diccionarios de Valves (ver patrón existente)
- Type hints en todos los métodos (`-> str`, `-> List[dict]`, `-> tuple[bool, str]`)
- Docstrings en todos los métodos públicos y privados con lógica no trivial

### Comentarios y secciones
- Usar separadores de sección con el patrón existente:
  ```python
  # ==========================================================================
  # NOMBRE DE SECCIÓN
  # ==========================================================================
  ```
- Los comentarios explican el **POR QUÉ**, no el qué
- Mantener los comentarios de seguridad existentes — son documentación crítica

### OpenAI Responses API
- Usar siempre `client.responses.create()` (NO `client.chat.completions.create`)
- Acceder al texto con `response.output_text`
- Loggear siempre `response.model`, `response.id` y tokens de uso

---

## 8. Convenciones de Git

### Ramas
```
tipo/descripcion-en-kebab-case-en-español

Ejemplos:
  feature/agente-sql-vacaciones
  feature/observabilidad-boletas
  fix/reconexion-postgresql-timeout
  fix/whitelist-keyword-faltante
  docs/schema-vista-presupuesto
  refactor/extraer-clase-base-pipeline
  chore/actualizar-dependencias
```

### Commits — Conventional Commits en español
```
tipo(agente o módulo): descripción en minúsculas sin punto final

Tipos permitidos:
  feat      → nueva funcionalidad
  fix       → corrección de bug
  docs      → solo documentación
  refactor  → mejora interna sin cambio de comportamiento
  test      → agregar o corregir pruebas
  chore     → mantenimiento, dependencias, configuración
  security  → cambios relacionados con seguridad

Ejemplos correctos:
  feat(presupuesto): agregar validación de vista autorizada en whitelist
  feat(boletas): implementar reconexión automática tras timeout
  fix(presupuesto): corregir extracción de tokens en Responses API
  fix(boletas): manejar valor None en columna FechaEliminacion
  docs(presupuesto): documentar lógica de enrutamiento de vistas
  refactor(pipeline): extraer _format_markdown_table a utilidad compartida
  security: agregar DBLINK a lista de keywords prohibidos en PostgreSQL
  chore: actualizar psycopg2-binary a 2.9.9
```

**Reglas:**
- Un commit = un cambio lógico (atómico)
- Nunca mezclar cambios de seguridad con refactoring en el mismo commit
- Si el cambio toca la whitelist SQL, siempre usar tipo `security`
- Máximo 72 caracteres en la primera línea
- Agregar cuerpo del commit cuando el cambio afecte el comportamiento de seguridad

---

## 9. Reglas para Aplicar Cambios

### Antes de modificar cualquier archivo
- [ ] Leer el archivo completo, no solo la sección a modificar
- [ ] Identificar el tipo de agente (SQL Server vs PostgreSQL) y respetar su dialecto
- [ ] Verificar que el cambio no rompe el flujo de `pipe()` ni la validación SQL
- [ ] Si el cambio toca `_validate_sql()` o `SQL_FORBIDDEN_KEYWORDS`, pedir confirmación explícita

### Al modificar un agente existente
- Mantener la estructura de secciones con sus separadores `=====`
- No cambiar el nombre de métodos del contrato OpenWebUI: `pipe()`, `on_startup()`, `on_shutdown()`
- No cambiar los prefijos de variables de entorno (rompe la configuración del contenedor)
- Si se agrega un método nuevo, ubicarlo en la sección que corresponde según el patrón
- Actualizar el docstring del archivo (cabecera) si cambia el comportamiento del agente

### Al crear un agente nuevo
1. Copiar la estructura del agente más similar (SQL Server → boletas, PostgreSQL → presupuesto)
2. Reemplazar: `self.name`, prefijo de variables de entorno, `SCHEMA_COLUMNS` o lógica de schema
3. Crear su `.env.<nombre>.example` correspondiente en `env/`
4. Crear su documentación en `docs/vistas/<nombre>.md`
5. Actualizar `README.md` con el nuevo agente en la lista
6. El primer commit del agente nuevo debe ser: `feat(<nombre>): agregar pipeline inicial`

### Al modificar SCHEMA_COLUMNS (solo agentes SQL Server)
- Es la única fuente de verdad del schema para el LLM en agentes SQL Server
- Cada columna debe tener: nombre, tipo entre paréntesis y comentario después de `--`
- El comentario debe explicar el significado de negocio, no solo repetir el nombre
- Ejemplo correcto:
  ```python
  - StatusBoleta (varchar)  -- Estado actual de la boleta (Activo, Eliminado)
  ```

### Al tocar el system_prompt del LLM
- Es código crítico — un cambio aquí afecta directamente la seguridad del agente
- Nunca eliminar las reglas de `PROMPT INJECTION DEFENSE` ni `ALLOWED STATEMENTS`
- Si se agregan reglas nuevas, mantener la numeración existente o renumerar todo
- Hacer commit separado solo para cambios de system_prompt con tipo `security` o `feat`

---

## 10. Archivos Restringidos

```
❌ NUNCA modificar sin confirmación explícita:
├── env/*.example              ← Pueden contener nombres de vars críticas de producción
├── Cualquier archivo .env     ← Nunca deben existir en el repo (están en .gitignore)
│
⚠️ Modificar con precaución — confirmar antes:
├── SQL_FORBIDDEN_KEYWORDS     ← Cambios afectan la seguridad de todos los agentes
├── SQL_REQUIRED_START         ← Cambio crítico de seguridad
├── system_prompt en           ← Afecta comportamiento y seguridad del LLM
│   _generar_sql_openai()
├── _validate_sql()            ← Núcleo de seguridad del pipeline
└── Valves (nombres de vars)   ← Rompe configuración del contenedor en producción
```

---

## 11. Variables de Entorno por Agente

### Boletas Worki (SQL Server)
```bash
# .env.boletas_worki.example
BOLETAS_DB_HOST=
BOLETAS_DB_PORT=1433
BOLETAS_DB_USER=
BOLETAS_DB_PASSWORD=
BOLETAS_DB_DATABASE=
BOLETAS_DB_VIEW=
BOLETAS_TEXT_TO_SQL_MODEL=gpt-4o
OPENAI_API_KEY_DISA=
```

### Presupuesto (PostgreSQL)
```bash
# .env.presupuesto.example
PRESUPUESTO_DB_HOST=
PRESUPUESTO_DB_PORT=5432
PRESUPUESTO_DB_USER=
PRESUPUESTO_DB_PASSWORD=
PRESUPUESTO_DB_DATABASE=
PRESUPUESTO_DB_SCHEMA=gold
PRESUPUESTO_DB_VIEW=presupuesto_mensual
PRESUPUESTO_DB_VIEW_ANUAL=resumen_anual
PRESUPUESTO_TEXT_TO_SQL_MODEL=gpt-4o
# Observabilidad
OBS_DB_HOST=
OBS_DB_PORT=5432
OBS_DB_DATABASE=observabilidad_db
OBS_DB_USER=
OBS_DB_PASSWORD=
OPENAI_API_KEY_DISA=
```

> Regla: cada agente nuevo tiene su propio bloque de variables con su prefijo único.
> La variable `OPENAI_API_KEY_DISA` es compartida entre todos los agentes.

---

## 12. Observabilidad — Aplica a TODOS los agentes

Todos los pipelines deben registrar cada ejecución en la BD de observabilidad.
Esta BD es **siempre PostgreSQL** (`observabilidad_db.observabilidad.agent_log`),
independientemente del DBMS de negocio del agente (SQL Server o PostgreSQL).

> Agentes que aún no tienen observabilidad implementada (ej: boletas_worki)
> deben recibirla como mejora planificada. Al implementarla, seguir
> exactamente el patrón del agente de presupuesto.

### Arquitectura de observabilidad

```
Agente (cualquier DBMS)
    │
    ├── conn        → BD de negocio (SQL Server o PostgreSQL según el agente)
    └── obs_conn    → BD de observabilidad (SIEMPRE PostgreSQL)
                      host: OBS_DB_HOST
                      db:   observabilidad_db
                      user: obs_writer (solo permisos INSERT)
                      tabla: observabilidad.agent_log
```

### Variables de entorno requeridas (todos los agentes)

Agregar a cada `.env.<nombre>.example`:
```bash
# Observabilidad (BD PostgreSQL compartida entre todos los agentes)
OBS_DB_HOST=
OBS_DB_PORT=5432
OBS_DB_DATABASE=observabilidad_db
OBS_DB_USER=
OBS_DB_PASSWORD=
```

### Componentes obligatorios por pipeline

```python
# 1. Dataclass de evento — acumula datos durante pipe()
@dataclass
class AgentEvent:
    pipeline_id:         str  = ""
    pipeline_version:    str  = ""
    session_id:          Optional[str]      = None
    turn_number:         Optional[int]      = None
    user_id:             Optional[str]      = None
    user_message:        Optional[str]      = None
    timestamp_inicio:    Optional[datetime] = None
    timestamp_fin:       Optional[datetime] = None
    modelo_llm:          Optional[str]      = None
    tokens_input:        Optional[int]      = None
    tokens_output:       Optional[int]      = None
    costo_usd_estimado:  Optional[float]    = None
    sql_generado:        Optional[str]      = None
    vista_utilizada:     Optional[str]      = None
    intento_numero:      int                = 1
    sql_validado:        Optional[bool]     = None
    razon_rechazo_sql:   Optional[str]      = None
    bd_duracion_ms:      Optional[int]      = None
    bd_filas_retornadas: Optional[int]      = None
    bd_error:            bool               = False
    query_plan:          Optional[dict]     = None
    resultado_tipo:      Optional[str]      = None
    respuesta_exitosa:   Optional[bool]     = None
    mensaje_error_interno: Optional[str]   = None

# 2. Constantes de costo LLM (actualizar si cambian precios en openai.com/pricing)
LLM_PRECIO_INPUT_POR_MTOKEN  = 2.00   # USD por 1M tokens de entrada
LLM_PRECIO_OUTPUT_POR_MTOKEN = 8.00   # USD por 1M tokens de salida

# 3. Conexión independiente a obs (en Valves y en __init__)
self.obs_conn = None   # separada de self.conn

# 4. Estructura de pipe() con observabilidad
def pipe(self, ...):
    evento = AgentEvent(pipeline_id=..., timestamp_inicio=datetime.now())
    respuesta_final = ""
    try:
        # ... lógica del agente, llenando evento ...
    except Exception as e:
        # ... manejo de errores, llenando evento ...
    finally:
        evento.timestamp_fin = datetime.now()
        try:
            self._registrar_evento(evento)   # fire-and-forget
        except Exception as e_obs:
            self._log(f"Error crítico en observabilidad: {e_obs}")
    return respuesta_final
```

### Reglas invariantes de observabilidad

- El registro es **fire-and-forget**: un fallo de observabilidad **nunca** interrumpe
  la respuesta al usuario ni lanza excepciones al caller
- `_registrar_evento()` y `_ensure_obs_connection()` tienen su propio `try/except`
- `AgentEvent` se instancia al **inicio** de `pipe()`, antes de cualquier lógica
- El bloque `finally` garantiza el registro incluso cuando `pipe()` lanza excepción
- Si se agrega un campo nuevo a `AgentEvent`, debe agregarse también al `INSERT` SQL
- El usuario `obs_writer` tiene permisos solo de `INSERT` — nunca usar credenciales
  de la BD de negocio para escribir en observabilidad
- `query_plan` se serializa a JSON antes de insertar: `json.dumps(evento.query_plan)`

---

## 13. Cambios Transversales — Mejoras que aplican a todos los agentes

Algunos cambios no son específicos de un agente sino que deben propagarse
a todos los pipelines del repositorio. Claude debe tratarlos como una
campaña de cambios coordinada, no como ediciones aisladas.

### Identificar si un cambio es transversal

Un cambio es transversal cuando:
- Afecta el **patrón arquitectónico común** (sección 4)
- Es una nueva funcionalidad que **todos los agentes deben tener** (ej: observabilidad)
- Corrige un **problema de seguridad** presente en múltiples agentes
- Estandariza algo que hoy está inconsistente entre agentes

### Cómo ejecutar un cambio transversal

```
1. Implementar el cambio en UN agente piloto (el más completo o el más similar)
2. Validar que funciona correctamente en ese agente
3. Propagar el cambio a los demás agentes, uno por uno
4. Actualizar CLAUDE.md para reflejar el nuevo estándar
5. Un commit por agente modificado (no un commit que toque todos a la vez)
```

### Commits para cambios transversales

```bash
# Commit del agente piloto
feat(presupuesto): implementar observabilidad como pipeline de referencia

# Commits de propagación
feat(boletas): agregar observabilidad siguiendo patrón de presupuesto
feat(<nombre>): agregar observabilidad siguiendo patrón de presupuesto

# Commit de documentación — SIEMPRE al final
docs(claude): actualizar CLAUDE.md — observabilidad aplica a todos los agentes
```

### Cambios transversales planificados

| Mejora | Estado | Agente de referencia |
|---|---|---|
| Observabilidad (`AgentEvent` + `agent_log`) | ⏳ Pendiente en boletas_worki | presupuesto |
| `pipeline_version` en `__init__` | ⏳ Pendiente en boletas_worki | presupuesto |
| `_ensure_connection()` con ping de reconexión | ⏳ Pendiente en boletas_worki | presupuesto |

> Al completar una mejora transversal, actualizar esta tabla cambiando
> el estado a ✅ Completado y hacer commit:
> `docs(claude): marcar [nombre mejora] como completada en tabla transversal`

---

## 15. Contexto de Despliegue

```
Entorno desarrollo:
  - Variables desde archivo .env.<agente> (cargado con python-dotenv)
  - override=False → el entorno del SO tiene prioridad sobre .env

Entorno producción (contenedor):
  - Variables inyectadas directamente en el entorno del contenedor
  - El código no cambia entre dev y prod — solo cambia el origen de las vars

OpenWebUI Pipeline Server:
  - Llama a on_startup() al iniciar el contenedor
  - Llama a pipe() en cada mensaje del usuario
  - Llama a on_shutdown() al detener el contenedor
  - Valves se configuran también desde la UI de OpenWebUI (sobrescriben env vars)
```

---

## 16. Infraestructura (futuro)

Cuando se agreguen archivos de infraestructura al repositorio:
- Ubicarlos en `infra/` en la raíz
- Estructura sugerida:
  ```
  infra/
  ├── docker-compose.yml
  ├── Dockerfile.pipeline
  └── README.md
  ```
- Commits de infra: `chore(infra): descripción del cambio`
- Los `Dockerfile` nunca deben contener credenciales ni copiar archivos `.env`

---

## 17. Mantenimiento de este archivo (CLAUDE.md)

### ¿Quién actualiza CLAUDE.md?

**Tú (Sergio) defines las reglas. Claude Code las ejecuta y puede proponer actualizaciones,
pero nunca debe modificar CLAUDE.md de forma autónoma sin confirmación explícita.**

El flujo correcto es:

```
Tú defines o cambias una convención
        ↓
Le indicas a Claude Code qué cambió
        ↓
Claude Code propone el texto actualizado para CLAUDE.md
        ↓
Tú lo revisas y apruebas
        ↓
Claude Code aplica el cambio con su commit correspondiente
```

### Cuándo debe actualizarse CLAUDE.md

| Situación | Quién detecta | Acción |
|---|---|---|
| Decides cambiar una convención de código o Git | Tú | Instruyes a Claude para actualizar la sección correspondiente |
| Se agrega un agente nuevo al repo | Claude (al crearlo) | Propone actualizar sección 3 (estructura) y sección 11 (variables de entorno) |
| Se completa un cambio transversal | Claude (al terminarlo) | Propone marcar la mejora como ✅ en la tabla de la sección 13 |
| Claude detecta que una regla del CLAUDE.md ya no refleja el código real | Claude | Señala la inconsistencia y propone la corrección — no la aplica sola |
| Cambian los precios de OpenAI | Tú | Actualizas las constantes `LLM_PRECIO_*` en la sección 12 |

### Reglas para modificar CLAUDE.md

- Nunca eliminar secciones completas — marcarlas como obsoletas si ya no aplican
- Los cambios a reglas de seguridad (sección 6) requieren confirmación explícita tuya
- Cada actualización de CLAUDE.md tiene su propio commit:
  ```bash
  docs(claude): [descripción del cambio en la regla o sección]

  # Ejemplos:
  docs(claude): agregar agente sql-vacaciones a estructura del repositorio
  docs(claude): actualizar precio LLM a gpt-4o-mini en sección observabilidad
  docs(claude): marcar pipeline_version como completada en tabla transversal
  docs(claude): agregar regla de reconexión a sección de cambios transversales
  ```
- CLAUDE.md **sí se versiona en Git** — cada cambio queda en el historial
