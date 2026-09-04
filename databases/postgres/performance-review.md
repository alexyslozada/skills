# Playbook: análisis de rendimiento de consultas en PostgreSQL

Método reutilizable para encontrar, en cualquier proyecto, qué consultas hay que mejorar y por
qué. Es el proceso que produjo `baseline-2026-08-26.md` y `review-2026-09-03.md` en este repo,
generalizado. Cada fase dice qué se busca, cómo se obtiene, qué señal indica un problema y qué
hacer con ella.

Principio que sostiene todo el método: **la base de datos dice qué duele, el código dice por
qué**. Ninguna de las dos fuentes alcanza sola. Una consulta de 0.06 ms que se ejecuta 49 millones
de veces no aparece en ningún `EXPLAIN`; un plan de 44 segundos no se explica sin saber quién lo
dispara y con qué parámetros.

---

## Fase 0 — Prerrequisitos

Sin esto no se puede empezar. Verificar todo antes de agendar el análisis.

### 0.1 Acceso a la base de datos de producción

- Usuario **de solo lectura** con permiso para leer `pg_stat_statements`, `pg_stat_user_tables`,
  `pg_stat_user_indexes`, `pg_settings` y ejecutar `EXPLAIN`. En PostgreSQL ≥ 10 basta con el rol
  `pg_read_all_stats`; en RDS/Aurora hay que otorgarlo explícitamente.
- Conexión directa (psql, DBeaver, un MCP de Postgres, etc.). Un túnel o bastión es suficiente;
  no hace falta acceso al host.
- Si hay varias bases en el mismo cluster (p. ej. una por microservicio), `pg_stat_statements` las
  ve todas desde cualquier conexión, pero `EXPLAIN` y las vistas `pg_stat_user_*` solo ven la base
  a la que estás conectado. Anotar a cuál corresponde cada tabla.
- Confirmar que se analiza **producción** o una réplica con tráfico real. Un entorno de desarrollo
  no tiene ni el volumen ni el patrón de llamadas que revela los problemas.

### 0.2 Estadísticas activas

```sql
SELECT extname, extversion FROM pg_extension WHERE extname = 'pg_stat_statements';
SELECT name, setting FROM pg_settings
WHERE name IN ('shared_preload_libraries', 'pg_stat_statements.max', 'pg_stat_statements.track',
               'track_io_timing', 'track_counts');
SELECT stats_reset, dealloc FROM pg_stat_statements_info;   -- PG ≥ 14
```

Qué se necesita:

- `pg_stat_statements` en `shared_preload_libraries` (requiere reinicio si no está) y
  `CREATE EXTENSION pg_stat_statements` en **cada** base que se quiera ver.
- `pg_stat_statements.track = top` (o `all` si se quieren funciones internas).
- `track_io_timing = on`: sin esto los planes no muestran `I/O Timings` y no se distingue CPU de
  disco.
- `stats_reset` dice desde cuándo acumulan los contadores. Si es de hace un año, los promedios
  mezclan estados viejos de la base con el actual (ver Fase 1.3).
- `dealloc` alto respecto a `pg_stat_statements.max` significa que se están perdiendo sentencias:
  las menos frecuentes se desalojan. En RDS/Aurora la base `rdsadmin` suele ocupar miles de
  entradas; subir `max` a 10 000 evita el problema.
- Si la extensión acaba de instalarse, **esperar al menos un ciclo completo de negocio** (una
  semana, para capturar cron semanales y picos) antes de analizar.

### 0.3 Acceso al código

- Repositorio del backend con permiso de lectura y el commit **desplegado en producción**
  identificado (`git log` del tag o de la rama de despliegue). Las consultas registradas en
  `pg_stat_statements` corresponden al código que corre, no al de la rama principal; si difieren,
  hay que saberlo antes de buscar.
- Cómo se generan las consultas: SQL crudo, query builder, ORM. Determina cómo buscar el texto
  (Fase 5).
- Dónde viven los disparadores no HTTP: cron, consumidores de cola, ETL, jobs. Y dónde está la
  capa de caché (Redis, memoria) y sus TTL.
- Acceso a la configuración de infraestructura que dispara jobs (cron de Kubernetes, scheduler
  externo, Cloud Scheduler) para saber periodicidades.

### 0.4 Herramientas

- Cliente SQL. Guardar cada resultado en archivos con fecha: el análisis se hace comparando.
- `grep`/`ripgrep` y la navegación del lenguaje (gopls, LSP) para seguir llamadas.
- Un lugar para escribir: el informe se construye mientras se investiga, no al final.

### 0.5 Consultar la base con un MCP de Postgres desde Claude Code

Todo el análisis se puede hacer sin salir del agente si la base está expuesta como servidor MCP.
Es lo que se usó en este proyecto.

**Configuración** (`.mcp.json` en la raíz del proyecto, **ignorado por git**; nunca versionar la
cadena de conexión):

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres",
               "postgresql://USUARIO:PASSWORD@HOST:5432/BASE_OLTP"]
    },
    "postgres-olap": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres",
               "postgresql://USUARIO:PASSWORD@HOST:5432/BASE_OLAP"]
    }
  }
}
```

Un servidor por base de datos: `EXPLAIN` y las vistas `pg_stat_user_*` solo ven la base de la
conexión. Caracteres especiales de la contraseña van URL-encoded (`*` → `%2A`, `$` → `%24`,
`@` → `%40`). Usar el usuario de solo lectura de 0.1.

**Cómo se invoca.** Cada servidor expone una herramienta `query` con un único parámetro `sql`.
Desde Claude Code el nombre completo es `mcp__<servidor>__query`. Si la herramienta aparece
diferida, primero se carga:

```
ToolSearch  query: "select:mcp__postgres__query,mcp__postgres-olap__query"
```

y después cada consulta se ejecuta así (es literalmente la llamada que hace el agente):

```
mcp__postgres__query
  sql: "SELECT current_database(), now(), (SELECT stats_reset FROM pg_stat_statements_info)"
```

En lo que sigue, **cada bloque `sql` de este documento se ejecuta tal cual como el parámetro
`sql` de `mcp__postgres__query`** (o del servidor que corresponda). El Apéndice C tiene el kit
completo en el orden en que se usa.

**Restricciones del servidor `@modelcontextprotocol/server-postgres`** que conviene conocer:

- Ejecuta en una transacción `READ ONLY`: `SELECT`, `EXPLAIN`, `EXPLAIN ANALYZE` de un `SELECT`
  y funciones de solo lectura funcionan; `pg_stat_statements_reset()`, `ANALYZE`, `VACUUM`,
  `CREATE INDEX` y cualquier DML fallan. Para eso hace falta psql o la migración.
- Una sentencia por llamada. Sin metacomandos de psql: `\d tabla` se reemplaza por
  `information_schema.columns` y `pg_indexes` (ver Apéndice C).
- El resultado vuelve como JSON; las columnas `bigint` llegan como texto. Redondear en SQL
  (`round(...)`, `left(query, 250)`) para que la salida quepa y sea legible.
- Sin tiempo máximo configurable: una consulta de 44 s se espera. No lanzar `EXPLAIN ANALYZE`
  sobre un Seq Scan de una tabla de GB sin haber estimado antes cuánto tardará.

---

## Fase 1 — Línea base: qué duele

Objetivo: una lista corta de consultas ordenadas por costo total, con contexto suficiente para
clasificarlas. Guardar el resultado íntegro (es el baseline contra el que se medirá después).

### 1.1 Reparto por base de datos

```sql
SELECT d.datname,
       round(sum(s.total_exec_time)/1000)::bigint AS total_s,
       sum(s.calls) AS calls, count(*) AS statements
FROM pg_stat_statements s JOIN pg_database d ON d.oid = s.dbid
GROUP BY 1 ORDER BY 2 DESC;
```

Dice dónde está el tiempo y si hay bases secundarias que merecen su propio análisis
(en este proyecto, `shorts` concentraba 71 000 s en tres sentencias).

### 1.2 Top 30 por tiempo total

```sql
SELECT queryid,
       round(total_exec_time/1000)::bigint AS total_s,
       calls,
       round(mean_exec_time::numeric, 3) AS mean_ms,
       round(min_exec_time::numeric, 3) AS min_ms,
       round(max_exec_time::numeric)    AS max_ms,
       rows,
       rows / GREATEST(calls, 1)        AS rows_per_call,
       (shared_blks_hit + shared_blks_read) / GREATEST(calls, 1) AS blks_per_call,
       round(100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0), 1) AS hit_pct,
       left(regexp_replace(query, '\s+', ' ', 'g'), 250) AS q
FROM pg_stat_statements
WHERE dbid = (SELECT oid FROM pg_database WHERE datname = current_database())
ORDER BY total_exec_time DESC
LIMIT 30;
```

Repetir ordenando por `calls`, por `rows`, por `shared_blks_read` (I/O real) y por
`mean_exec_time` con `calls > 100`. Cada orden revela una familia distinta de problemas:

| Orden | Qué revela |
|---|---|
| `total_exec_time` | Dónde se va el tiempo del cluster. Es la lista principal. |
| `calls` | Consultas por petición: middlewares, N+1, endpoints públicos sin caché. |
| `rows` / `rows_per_call` | Consultas que traen miles de filas para agregar en la aplicación. |
| `shared_blks_read` | Lo que realmente toca disco. En Aurora es lo que cuesta dinero. |
| `mean_exec_time` | Reportes, ETL y planes malos. Filtrar `calls > 100` para evitar ruido. |

Guardar el texto completo de las consultas del top: `SELECT query FROM pg_stat_statements WHERE
queryid = …`. Se necesita en la Fase 5.

### 1.3 Leer los números con cuidado

- `min_exec_time` es el mejor plan que la consulta tuvo alguna vez. Si `min` es 0.007 ms y
  `mean` es 20 ms, o la consulta depende mucho del parámetro (usuario con muchas filas vs pocas)
  o el plan cambió en algún momento (se creó un índice). Un `min` bajo con `mean` alto después de
  una corrección significa que el promedio arrastra historia vieja: hay que medir por delta.
- `rows_per_call` > 100 en una consulta de aplicación (no de reporte) casi siempre es agregación
  en código o falta de paginación.
- `blks_per_call` alto con `rows_per_call` bajo es un Seq Scan o un índice que no filtra.
- `hit_pct` bajo señala tablas que no caben en `shared_buffers`; el costo está en I/O.
- Una consulta con `calls` altísimas y `mean` de microsegundos **sigue siendo un problema** si se
  ejecuta por petición: cada una es un round-trip de red completo.

### 1.4 Si los contadores no se resetearán: medir por delta

Cuando el snapshot anterior existe, comparar `calls` y `total_exec_time` actuales contra los
guardados. La diferencia describe **exactamente el período entre snapshots**, sin contaminación
de estados anteriores:

```
ms_por_llamada_reciente = (total_ms_ahora − total_ms_antes) / (calls_ahora − calls_antes)
```

Es la única forma de confirmar que un índice nuevo funciona mientras el promedio acumulado sigue
alto. Alternativa: `SELECT pg_stat_statements_reset()` después de guardar el baseline y volver a
leer a los 7 días. Resetear pierde la historia, así que guardar primero.

---

## Fase 2 — Tablas e índices: dónde faltan índices sin mirar consultas

Esta fase encuentra problemas que `pg_stat_statements` no muestra porque las sentencias fueron
desalojadas o vienen de otro cliente.

### 2.1 Seq Scans sobre tablas grandes

```sql
SELECT relname, n_live_tup,
       pg_size_pretty(pg_relation_size(relid)) AS heap,
       seq_scan, seq_tup_read, idx_scan,
       round(seq_tup_read::numeric / GREATEST(seq_scan, 1)) AS tup_per_seq
FROM pg_stat_user_tables
WHERE pg_relation_size(relid) > 20 * 1024 * 1024 AND seq_scan > 10000
ORDER BY seq_tup_read DESC LIMIT 25;
```

Señal: `seq_scan` de cientos de miles con `tup_per_seq` del tamaño de la tabla. Cada fila es una
tabla a la que le falta un índice para algún predicado. Si no aparece ninguna sentencia que la
explique, el scanner es otro cliente (app legacy, script) o fue desalojado (Fase 0.2).

### 2.2 Índices existentes por tabla

```sql
SELECT tablename, indexname, indexdef FROM pg_indexes
WHERE schemaname = 'public' AND tablename IN ('…') ORDER BY 1, 2;
```

Para cada consulta del top, comprobar que existe un índice **cuya primera columna** sea el
filtro de igualdad más selectivo y cuyas columnas siguientes cubran el rango y el `ORDER BY`. Un
índice `(a, b)` no sirve para `WHERE b = $1`. Un índice sobre `created_at` no sirve para
`WHERE created_at::date = $1`.

### 2.3 Índices sin uso, duplicados y su tamaño

```sql
SELECT relname, indexrelname,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size,
       idx_scan, idx_tup_read
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC LIMIT 40;
```

- `idx_scan` cercano a 0 tras meses: candidato a eliminar (verificar que no sea un UNIQUE que
  sostiene una restricción).
- Dos índices con la misma definición (frecuente: `_uk` y `_ix` sobre la misma columna): uno tiene
  0 scans porque el planner elige arbitrariamente el otro. Eliminar el no-único.
- Un índice `(a)` cuando existe `(a, b)`: redundante, salvo que `(a, b)` sea mucho más grande.
- `idx_tup_read / idx_scan` enorme (miles de tuplas leídas por scan) delata un índice que se
  recorre entero porque la condición cae en una columna no líder.

### 2.4 Bloat, TOAST y vacuum

```sql
SELECT relname,
       pg_size_pretty(pg_total_relation_size(relid)) AS total,
       pg_size_pretty(pg_relation_size(relid))       AS heap,
       pg_size_pretty(pg_indexes_size(relid))        AS idx,
       n_live_tup, n_dead_tup, n_tup_ins, n_tup_upd, n_tup_hot_upd,
       last_autovacuum, last_autoanalyze, autovacuum_count
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC LIMIT 20;
```

Señales:

- Índices que pesan varias veces más que el heap (esperado para un btree de int4: ~20–25 bytes
  por entrada). Reindexar con `REINDEX INDEX CONCURRENTLY`.
- `n_tup_upd` ≫ `n_tup_ins` con `last_autovacuum` de hace meses o años: el umbral por defecto del
  20 % nunca se alcanza en tablas grandes. Bajar `autovacuum_vacuum_scale_factor` por tabla.
- `last_autoanalyze` viejo en tablas que crecen: el planner estima con datos viejos
  (ver Fase 4, estimaciones vs. real).
- Tabla con pocas filas pero cientos de MB: es TOAST. Comprobar con
  `pg_relation_size(reltoastrelid)` y `pg_column_size(columna)` por fila. Suele ser una tabla de
  "caché" con JSON grandes reescrita con frecuencia.
- Tablas que solo crecen (logs, tokens, códigos efímeros): comprobar cuántas filas ya no sirven
  (`WHERE expires_at < now()`, `created_at < now() − interval '90 days'`) y si existe alguna purga.

### 2.5 Parámetros del planner

```sql
SELECT name, setting, unit FROM pg_settings
WHERE name IN ('shared_buffers', 'effective_cache_size', 'work_mem', 'random_page_cost',
               'seq_page_cost', 'max_parallel_workers_per_gather', 'server_version');
```

`random_page_cost = 4` en almacenamiento SSD (todo RDS/Aurora) inclina al planner hacia Seq
Scans; `1.1` es el valor habitual. No cambiarlo sin medir, pero anotarlo.

---

## Fase 3 — Clasificar cada consulta del top

Antes de abrir un `EXPLAIN`, decidir a qué familia pertenece. La familia determina la
herramienta:

| Familia | Cómo se reconoce | Herramienta principal |
|---|---|---|
| **Plan malo** | `mean_ms` alto o `blks_per_call` alto con pocas filas | `EXPLAIN (ANALYZE, BUFFERS)` (Fase 4) |
| **Por petición** | `calls` del orden del tráfico total; `mean` de microsegundos | Buscar el middleware o el hook (Fase 5) |
| **N+1** | `calls` proporcional a otra consulta × N; se ejecuta con IDs uno a uno | Buscar el loop (Fase 5) |
| **Trae todo para agregar** | `rows_per_call` en cientos o miles fuera de reportes | Mover la agregación a SQL |
| **Endpoint público sin caché** | `calls` altas sobre datos que cambian poco (precios, catálogo, contadores) | Caché con TTL corto |
| **Batch / ETL / cron** | Pocas `calls` con `mean` en cientos de ms; rangos `BETWEEN` | Índice sobre la columna de rango; revisar ventana |
| **Escritura lenta** | `INSERT`/`UPDATE` con `mean` > 2–3 ms | Contar índices y triggers de la tabla; bloat |

Una consulta puede estar en dos familias (plan malo **y** por petición). Anotarlo: la corrección
son dos cambios.

---

## Fase 4 — `EXPLAIN (ANALYZE, BUFFERS)` con parámetros reales

### 4.1 Elegir los parámetros

`pg_stat_statements` guarda `$1, $2…`, no los valores. Hay que construir el peor caso realista:

- Para filtros por usuario/curso/entidad: el que tiene más filas.
  `SELECT user_id FROM tabla GROUP BY 1 ORDER BY count(*) DESC LIMIT 1`. (Ojo: esa subconsulta es
  un Seq Scan; en tablas grandes ejecutarla una vez y copiar el valor.)
- Para rangos de fecha: el rango que usa el código (día, mes, "últimas 24 h"), no uno arbitrario.
- Para enums y tipos propios: consultar `enum_range(NULL::tipo)` para no adivinar literales.
- Mirar la distribución, no solo el máximo:

  ```sql
  SELECT percentile_cont(0.5) WITHIN GROUP (ORDER BY n) AS p50,
         percentile_cont(0.9) WITHIN GROUP (ORDER BY n) AS p90,
         percentile_cont(0.99) WITHIN GROUP (ORDER BY n) AS p99, max(n)
  FROM (SELECT user_id, count(*) n FROM tabla GROUP BY 1) s;
  ```

  Si p50 es 6 y el máximo es 12 000, el promedio de `pg_stat_statements` lo dominan los usuarios
  activos, que son justamente los que más llaman.

### 4.2 Reglas de seguridad

- `EXPLAIN ANALYZE` **ejecuta** la consulta. Solo sobre `SELECT`. Para `INSERT/UPDATE/DELETE`
  usar `EXPLAIN` sin `ANALYZE`, o envolver en `BEGIN; EXPLAIN ANALYZE …; ROLLBACK;` en una sesión
  donde eso sea aceptable.
- Una consulta que en producción tarda 500 ms puede tardar 40 s en frío. Ejecutarla una vez es
  aceptable; no ejecutarla en loop.
- Usar una conexión de solo lectura para que un descuido no pueda escribir.

### 4.3 Qué buscar en el plan

| Señal en el plan | Diagnóstico | Corrección típica |
|---|---|---|
| `Seq Scan … Rows Removed by Filter: N` con N ≈ tamaño de la tabla | Falta índice para el predicado | Índice (parcial si el filtro es fijo, p. ej. `WHERE status = 'PUBLISHED'`) |
| `Index Scan using idx_(a,b) … Index Cond: (b = …)` | Índice usado por una columna no líder: se recorre entero | Índice con `b` primero |
| `Filter: ((col)::date = …)` o cualquier función sobre la columna | Cast/función anula el índice | Reescribir como rango `col >= d AND col < d + 1` o índice por expresión |
| `Nested Loop … loops=7062` bajo un `Sort` + `Limit` | Join de todas las filas **antes** de ordenar y cortar | Subconsulta con `ORDER BY … LIMIT` y join después; o índice `(filtro, orden)` |
| `rows=60` estimado vs `rows=7062` real | Estadísticas viejas o predicado sin estadística (funciones) | `ANALYZE`; índice por expresión; `CREATE STATISTICS` |
| `Index Only Scan … Heap Fetches: N` con N ≈ filas | Visibility map desactualizado | `VACUUM` de la tabla; autovacuum más agresivo |
| `Buffers: shared read=` alto con `I/O Timings` dominante | Trabajo en disco, no en CPU | Reducir filas tocadas (índice) o reducir tamaño (bloat, columnas) |
| `Sort Method: external merge` | `work_mem` insuficiente para esa consulta | Índice que entregue el orden; o `work_mem` por sesión |
| `Parallel Seq Scan` con `Workers Launched: 2` en consulta OLTP | Consulta pequeña resolviéndose con fuerza bruta | Índice; el paralelismo esconde el costo de CPU |
| Subplan `(SELECT … FROM parameters WHERE name = …)` ejecutado por fila | Subconsulta correlacionada innecesaria | Resolverla una vez (`InitPlan`) o pasarla como parámetro |

Registrar para cada plan: tiempo, bloques (`hit`/`read`), el nodo culpable y la cita textual de
la línea. Es la evidencia que va en el informe.

---

## Fase 5 — Mapear cada consulta al código

Objetivo: para cada consulta saber **quién la ejecuta, desde qué entrada, con qué frecuencia, con
qué parámetros y si hay caché delante**.

### 5.1 Encontrar el origen

- SQL crudo: buscar un fragmento distintivo del texto (`grep -rn "length(trim(" internal/`).
  Normalizar espacios: `pg_stat_statements` compacta el texto.
- Query builder / ORM: buscar por nombre de tabla y por los campos del `WHERE`. En builders con
  mapa campo → columna (como `sqlcraft` aquí), buscar el mapa: es donde aparecen expresiones como
  `"created_at_date": "created_at::date"`.
- Si el texto no aparece en el código: el commit desplegado es distinto (columnas de más o de
  menos en el `SELECT` lo delatan) o la consulta viene de otro servicio/cliente.

### 5.2 Subir por la cadena de llamadas

Repositorio → use case → handler/middleware/consumidor → ruta o disparador. Anotar
`archivo:línea` en cada salto. Preguntas obligatorias:

1. **¿Está en un middleware o en un hook de autenticación?** Entonces se ejecuta en cada petición
   de cada ruta donde el middleware está registrado. Contar las rutas.
2. **¿Está dentro de un `for`?** N+1. Ver qué colección se recorre y si el repositorio ofrece
   una variante `IN (…)`.
3. **¿Es un endpoint público o sin autenticación?** Sin caché ni rate limit, sus `calls` son el
   tráfico de la página.
4. **¿La dispara un evento, cron o ETL?** Buscar la periodicidad real (tabla de logs de
   sincronización, configuración del scheduler), no la supuesta.
5. **¿Qué hace el código con las filas?** Si las recorre para sumar, contar, promediar o filtrar,
   la agregación debe ir en SQL.
6. **¿Hay caché?** Cuál es el TTL, qué clave, quién la invalida y **qué pasa en el fallback**: un
   fallback que reconstruye desde la base sin volver a escribir la caché convierte un miss
   permanente en una consulta por petición.
7. **¿Los parámetros que llegan son los que el índice espera?** Zonas horarias, unix vs
   timestamp, `::date` en la aplicación o en el SQL.

### 5.3 Patrones de código que producen consultas caras

Lista de verificación para leer el use case:

- Resultado de una consulta descartado (`_, err := repo.FindOne(…)`): consulta que solo valida
  existencia y podría ser un `EXISTS` o no existir.
- `SELECT + UPDATE` para "crear o acumular": debería ser `INSERT … ON CONFLICT DO UPDATE`.
  Además elimina la carrera entre dos eventos concurrentes.
- Tipo/rol del usuario resuelto contra la base cuando ya viaja en el token o en un header.
- Tablas de configuración (`parameters`, `endpoint_limits`, `feature_flags`) leídas por petición.
- Búsqueda de "todos los registros de X para calcular un resumen" en lugar de `GROUP BY`.
- Rangos por día con `::date`, `date_trunc`, `EXTRACT` sobre la columna en vez de sobre el
  parámetro.
- `LIMIT` aplicado después de un join 1:N o después de un join a una tabla ancha.
- Cachés generadas manualmente (un `PUT` administrativo) sin regeneración automática al cambiar
  los datos que contienen.

---

## Fase 6 — Verificar hipótesis con datos

Cada hallazgo debe poder confirmarse con una consulta o un plan, no con intuición. Ejemplos de
verificaciones que valieron la pena en este proyecto:

- "La caché no se regenera": `SELECT type, updated_at FROM general_data` mostró una entrada de
  hace 11 meses, y `SELECT count(*) FROM specialty_routes WHERE created_at > <esa fecha>` mostró la
  ruta que no estaba cacheada.
- "El ETL lee más de lo que debería": comparar `rows / calls` de la consulta fuente con las filas
  por día en la tabla destino (`GROUP BY created_at::date`); 39 000 por corrida horaria contra
  18 000 por día significa una ventana de dos días.
- "Ese job ya no corre": `SELECT etl_alias, state, count(*), max(created_at) FROM etl_sync_logs
  GROUP BY 1, 2`. Un job con 1 091 errores y 28 éxitos, el último hace seis meses, cambia la
  prioridad del hallazgo.
- "El índice nuevo se usa": `idx_scan` en `pg_stat_user_indexes` **y** el delta de ms/llamada
  (Fase 1.4). Uno solo no basta.
- "Los tokens expirados no se purgan": `count(*) FILTER (WHERE expires_at < now())` contra el
  total.
- "Cuántas entidades disparan el reporte": `SELECT count(DISTINCT user_id) FROM course_professor`
  explica 126 llamadas diarias sin necesidad de leer el scheduler.

---

## Fase 7 — Priorizar

Ordenar por **tiempo que se recuperará**, ponderado por riesgo del cambio. Regla práctica:

| Prioridad | Tipo de cambio | Riesgo | Ejemplos |
|---|---|---|---|
| 1 | Índice faltante (una migración, sin tocar código) | Bajo (`CONCURRENTLY` en tablas grandes) | Fase 2, Fase 4 |
| 2 | Acción operativa inmediata | Nulo | Regenerar una caché, mergear una rama ya escrita, `VACUUM` |
| 3 | Reescritura de una consulta (misma semántica) | Medio (probar con los mismos parámetros) | Quitar casts, subconsulta + limit, `GROUP BY` |
| 4 | Caché en endpoints y tablas de configuración | Medio (invalidación) | TTL 60 s en datos que cambian por hora |
| 5 | Mantenimiento en ventana | Medio-alto (locks) | `VACUUM FULL`, reindex, purgas, autovacuum por tabla, parámetros |
| 6 | Cambios de diseño | Alto | Acumulados materializados, mover cachés de la base a Redis |

Para cada elemento anotar el tiempo acumulado o el delta que lo justifica. Un hallazgo sin número
no entra en el plan.

Casos que suben de prioridad aunque su tiempo total sea modesto: consultas cuyo tiempo **crece**
entre snapshots (indican una funcionalidad nueva sin índice), consultas con `max_exec_time` de
decenas de segundos (bloquean conexiones del pool) y todo lo que esté en el camino del checkout
o del login.

---

## Fase 8 — Documentar

El informe se lee en el futuro para comparar. Formato mínimo por hallazgo:

1. **Consulta** (texto normalizado) y números: `total_s`, `calls`, `mean_ms`, delta si existe.
2. **Origen**: `archivo:línea` del repositorio, use case, ruta o disparador, frecuencia.
3. **Evidencia**: la línea del plan o la consulta de verificación, citada.
4. **Causa** en una frase.
5. **Corrección**: SQL del índice o de la reescritura, o el cambio de código, listo para copiar.
6. **Ganancia esperada** y cómo se verificará.

Y al principio del documento: fecha del snapshot, `stats_reset`, commit desplegado, si los
contadores se resetearon. Al final: orden de ejecución sugerido.

Mantener un archivo por snapshot (`baseline-<fecha>.md`, `review-<fecha>.md`) en el repo, junto
al código que corrige. Corregir en el documento nuevo lo que el anterior tenía mal (p. ej. un
supuesto sobre desde cuándo acumulan las estadísticas).

---

## Fase 9 — Medir después y repetir

1. Desplegar el primer bloque (índices).
2. Guardar un snapshot y, si se decide, `pg_stat_statements_reset()`.
3. A los 7 días: repetir Fase 1 y comparar por delta. Confirmar `idx_scan` de cada índice nuevo.
   Un índice que no se usa se elimina; no se deja "por si acaso".
4. Volver a la Fase 3 con el nuevo top: al quitar las consultas más caras, las siguientes suben
   y pueden cambiar de familia (una consulta "por petición" pasa a ser la primera cuando ya no la
   tapa el reporte lento).
5. Ciclo mensual hasta que el top 20 sea estable y todo lo que quede tenga una justificación
   escrita.

---

## Apéndice A — Checklist rápida

```
[ ] Acceso read-only a producción con pg_read_all_stats
[ ] MCP de Postgres configurado (un servidor por base) y verificado con C.0.1
[ ] pg_stat_statements instalado en cada base; track_io_timing = on
[ ] stats_reset y dealloc anotados; max suficiente
[ ] Commit desplegado identificado; repo clonado en ese commit
[ ] Snapshot guardado: por base, top 30 × 5 órdenes, texto completo de cada consulta
[ ] pg_stat_user_tables: seq_scan, tamaños, vacuum/analyze, dead tuples
[ ] pg_indexes + pg_stat_user_indexes: faltantes, duplicados, sin uso, bloat
[ ] Cada consulta del top clasificada en una familia
[ ] EXPLAIN (ANALYZE, BUFFERS) con peor caso real para las de plan malo
[ ] archivo:línea + disparador + frecuencia + caché para cada una
[ ] Cada hipótesis verificada con una consulta de datos
[ ] Informe con evidencia, SQL propuesto y orden
[ ] Fecha de la próxima medición agendada
```

## Apéndice B — Errores frecuentes

- Confiar en `mean_exec_time` después de crear un índice: el promedio acumulado tarda meses en
  moverse. Medir por delta.
- Ejecutar `EXPLAIN` sin `ANALYZE` y creer que el plan es el real: el planner puede elegir otro
  con los parámetros reales, y sin `ANALYZE` no hay `Rows Removed by Filter` ni `loops`.
- Ejecutar `EXPLAIN` con un parámetro "cualquiera": el usuario con 6 filas no muestra el problema
  del usuario con 12 000.
- Eliminar índices porque "solo se usan en joins": si la columna aparece en `JOIN … ON` es un
  filtro de igualdad para el lado interno del nested loop. Verificar contra `pg_stat_statements`
  antes de cada `DROP INDEX`, y volver a verificar `idx_scan` después.
- Asumir que una caché existe porque hay un `cache.Get`: leer el fallback.
- Asumir que un job corre porque está registrado: leer su tabla de logs.
- Crear índices sin `CONCURRENTLY` en tablas de GB dentro de una migración que se aplica en horario
  de tráfico. Verificar antes si la herramienta de migraciones envuelve el archivo en una
  transacción (`CONCURRENTLY` no puede ejecutarse dentro de una).
- Olvidar las bases secundarias del mismo cluster.
- Analizar sin guardar: sin baseline no hay forma de demostrar la mejora ni de detectar la
  regresión.

## Apéndice C — Kit de consultas para el MCP, en orden de uso

Cada entrada es una llamada a `mcp__postgres__query` (o al servidor indicado) con el texto de
`sql` exacto. Están numeradas en el orden en que se ejecutan durante un análisis. Sustituir lo
que va entre `<>`.

### C.0 Verificar el entorno

```
# C.0.1 — a qué base estoy conectado y desde cuándo acumulan las estadísticas
mcp__postgres__query
sql: SELECT current_database(), now(), (SELECT stats_reset FROM pg_stat_statements_info) AS stats_reset, (SELECT dealloc FROM pg_stat_statements_info) AS dealloc, (SELECT count(*) FROM pg_stat_statements) AS statements

# C.0.2 — parámetros relevantes del servidor
mcp__postgres__query
sql: SELECT name, setting, unit FROM pg_settings WHERE name IN ('server_version','shared_buffers','effective_cache_size','work_mem','random_page_cost','track_io_timing','pg_stat_statements.max','pg_stat_statements.track','max_parallel_workers_per_gather')
```

### C.1 Línea base

```
# C.1.1 — reparto por base de datos del cluster
mcp__postgres__query
sql: SELECT d.datname, round(sum(s.total_exec_time)/1000)::bigint AS total_s, sum(s.calls) AS calls, count(*) AS statements FROM pg_stat_statements s JOIN pg_database d ON d.oid = s.dbid GROUP BY 1 ORDER BY 2 DESC

# C.1.2 — top 30 por tiempo total (cambiar ORDER BY por calls / rows / shared_blks_read / mean_exec_time)
mcp__postgres__query
sql: SELECT queryid, round(total_exec_time/1000)::bigint AS total_s, calls, round(mean_exec_time::numeric,3) AS mean_ms, round(min_exec_time::numeric,3) AS min_ms, round(max_exec_time::numeric) AS max_ms, rows, rows/GREATEST(calls,1) AS rows_per_call, (shared_blks_hit+shared_blks_read)/GREATEST(calls,1) AS blks_per_call, left(regexp_replace(query, '\s+', ' ', 'g'), 250) AS q FROM pg_stat_statements WHERE dbid = (SELECT oid FROM pg_database WHERE datname = current_database()) ORDER BY total_exec_time DESC LIMIT 30

# C.1.3 — top por tiempo medio, solo consultas con volumen (reportes, ETL, planes malos)
mcp__postgres__query
sql: SELECT calls, round(total_exec_time/1000)::bigint AS total_s, round(mean_exec_time::numeric,1) AS mean_ms, rows/GREATEST(calls,1) AS rows_per_call, left(regexp_replace(regexp_replace(query, '\s+', ' ', 'g'), '^SELECT .*? FROM', 'SELECT … FROM'), 220) AS q FROM pg_stat_statements WHERE dbid = (SELECT oid FROM pg_database WHERE datname = current_database()) AND mean_exec_time > 100 AND calls > 200 ORDER BY total_exec_time DESC LIMIT 20

# C.1.4 — texto completo de una o varias consultas del top
mcp__postgres__query
sql: SELECT queryid, calls, round(mean_exec_time::numeric,2) AS mean_ms, regexp_replace(query, '\s+', ' ', 'g') AS q FROM pg_stat_statements WHERE queryid IN (<id1>, <id2>)

# C.1.5 — todas las consultas sobre una tabla (para diseñar un índice que sirva a todas)
mcp__postgres__query
sql: SELECT calls, round(total_exec_time/1000)::bigint AS total_s, round(mean_exec_time::numeric,2) AS mean_ms, (shared_blks_hit+shared_blks_read)/GREATEST(calls,1) AS blks_per_call, rows/GREATEST(calls,1) AS rows_per_call, left(regexp_replace(regexp_replace(query, '\s+', ' ', 'g'), '^SELECT .*? FROM', 'SELECT … FROM'), 260) AS q FROM pg_stat_statements WHERE dbid = (SELECT oid FROM pg_database WHERE datname = current_database()) AND query ~* 'FROM <tabla>( AS \w+| \w)? (WHERE|INNER|LEFT|JOIN)' AND calls > 50 ORDER BY total_exec_time DESC LIMIT 15

# C.1.6 — sumar una familia de consultas (variantes de IN (...) con distinto número de parámetros)
mcp__postgres__query
sql: SELECT sum(calls) AS calls, round(sum(total_exec_time)/1000)::bigint AS total_s, sum(rows) AS rows FROM pg_stat_statements WHERE dbid = (SELECT oid FROM pg_database WHERE datname = current_database()) AND query ~* 'FROM <tabla> WHERE <columna> IN \('

# C.1.7 — buscar una consulta por un fragmento de texto (útil para las que se salieron del top)
mcp__postgres__query
sql: SELECT queryid, calls, round(total_exec_time/1000)::bigint AS total_s, round(mean_exec_time::numeric,2) AS mean_ms, left(regexp_replace(query, '\s+', ' ', 'g'), 300) AS q FROM pg_stat_statements WHERE query ILIKE '%<fragmento>%' ORDER BY total_exec_time DESC LIMIT 10
```

### C.2 Tablas e índices

```
# C.2.1 — tablas grandes con muchos Seq Scan
mcp__postgres__query
sql: SELECT relname, n_live_tup, pg_size_pretty(pg_relation_size(relid)) AS heap, seq_scan, seq_tup_read, idx_scan, round(seq_tup_read::numeric/GREATEST(seq_scan,1)) AS tup_per_seq FROM pg_stat_user_tables WHERE pg_relation_size(relid) > 20*1024*1024 AND seq_scan > 10000 ORDER BY seq_tup_read DESC LIMIT 25

# C.2.2 — estado e índices de un conjunto de tablas (reemplaza a \d)
mcp__postgres__query
sql: SELECT t.relname, t.n_live_tup, pg_size_pretty(pg_total_relation_size(t.relid)) AS size, t.seq_scan, t.idx_scan, t.last_autovacuum::date, t.last_autoanalyze::date, (SELECT string_agg(indexdef, ' | ') FROM pg_indexes i WHERE i.tablename = t.relname AND i.schemaname = 'public') AS indexes FROM pg_stat_user_tables t WHERE t.relname IN ('<tabla1>','<tabla2>') ORDER BY t.relname

# C.2.3 — columnas y tipos de una tabla (reemplaza a \d)
mcp__postgres__query
sql: SELECT string_agg(column_name||':'||data_type, ', ' ORDER BY ordinal_position) FROM information_schema.columns WHERE table_name = '<tabla>'

# C.2.4 — valores de un enum (para no adivinar literales en el EXPLAIN)
mcp__postgres__query
sql: SELECT enum_range(NULL::<tipo_enum>)

# C.2.5 — índices por tabla con tamaño y uso
mcp__postgres__query
sql: SELECT relname, indexrelname, pg_size_pretty(pg_relation_size(indexrelid)) AS size, idx_scan, idx_tup_read FROM pg_stat_user_indexes WHERE relname IN ('<tabla1>','<tabla2>') ORDER BY relname, pg_relation_size(indexrelid) DESC

# C.2.6 — índices más grandes del esquema, con uso
mcp__postgres__query
sql: SELECT relname, indexrelname, pg_size_pretty(pg_relation_size(indexrelid)) AS size, idx_scan FROM pg_stat_user_indexes ORDER BY pg_relation_size(indexrelid) DESC LIMIT 40

# C.2.7 — tablas más grandes: heap vs índices
mcp__postgres__query
sql: SELECT relname, pg_size_pretty(pg_total_relation_size(relid)) AS total, pg_size_pretty(pg_relation_size(relid)) AS heap, pg_size_pretty(pg_indexes_size(relid)) AS idx, n_live_tup FROM pg_stat_user_tables ORDER BY pg_total_relation_size(relid) DESC LIMIT 15

# C.2.8 — vacuum, analyze y tuplas muertas de las tablas calientes
mcp__postgres__query
sql: SELECT relname, n_live_tup, n_dead_tup, n_tup_ins, n_tup_upd, n_tup_hot_upd, n_tup_del, last_autovacuum, last_vacuum, last_autoanalyze, last_analyze, autovacuum_count FROM pg_stat_user_tables WHERE relname IN ('<tabla1>','<tabla2>')

# C.2.9 — TOAST de una tabla (pocas filas, muchos MB)
mcp__postgres__query
sql: SELECT c.relname, pg_size_pretty(pg_relation_size(c.oid)) AS heap, pg_size_pretty(pg_relation_size(c.reltoastrelid)) AS toast, s.n_tup_upd, s.n_tup_hot_upd, s.last_autovacuum, (SELECT last_autovacuum FROM pg_stat_all_tables WHERE relid = c.reltoastrelid) AS toast_last_autovacuum FROM pg_class c JOIN pg_stat_user_tables s ON s.relid = c.oid WHERE c.relname = '<tabla>'

# C.2.10 — tamaño real por fila de una columna grande
mcp__postgres__query
sql: SELECT id, <columna_clave>, pg_size_pretty(pg_column_size(<columna_grande>)::bigint) AS size, updated_at FROM <tabla> ORDER BY pg_column_size(<columna_grande>) DESC

# C.2.11 — triggers de una tabla (explican INSERT/UPDATE lentos)
mcp__postgres__query
sql: SELECT c.relname, t.tgname, pg_get_triggerdef(t.oid) FROM pg_trigger t JOIN pg_class c ON c.oid = t.tgrelid WHERE NOT t.tgisinternal AND c.relname IN ('<tabla1>','<tabla2>') ORDER BY 1, 2
```

### C.3 Planes

```
# C.3.1 — peor caso realista: la entidad con más filas (ejecutar UNA vez; es un Seq Scan)
mcp__postgres__query
sql: SELECT <columna_filtro>, count(*) FROM <tabla> GROUP BY 1 ORDER BY 2 DESC LIMIT 1

# C.3.2 — distribución de filas por entidad
mcp__postgres__query
sql: SELECT count(*) AS entidades, percentile_cont(0.5) WITHIN GROUP (ORDER BY n) AS p50, percentile_cont(0.9) WITHIN GROUP (ORDER BY n) AS p90, percentile_cont(0.99) WITHIN GROUP (ORDER BY n) AS p99, max(n) FROM (SELECT <columna_filtro>, count(*) n FROM <tabla> GROUP BY 1) s

# C.3.3 — plan real con buffers (solo SELECT; sustituir los $n por los valores de C.3.1)
mcp__postgres__query
sql: EXPLAIN (ANALYZE, BUFFERS) SELECT <columnas> FROM <tabla> WHERE <predicado con valores reales> ORDER BY <orden> LIMIT <n>

# C.3.4 — plan sin ejecutar (para INSERT/UPDATE/DELETE o consultas que se sabe que tardan minutos)
mcp__postgres__query
sql: EXPLAIN (BUFFERS, FORMAT TEXT) <sentencia>
```

### C.4 Verificaciones de hipótesis (ejemplos reales; adaptar)

```
# C.4.1 — ¿desde cuándo no se regenera una caché guardada en la base?
mcp__postgres__query
sql: SELECT type, updated_at FROM <tabla_cache> ORDER BY updated_at

# C.4.2 — ¿qué entidades nacieron después de esa regeneración?
mcp__postgres__query
sql: SELECT id, name, created_at FROM <tabla> WHERE created_at > '<fecha_regeneracion>'

# C.4.3 — ¿corre ese job y con qué resultado?
mcp__postgres__query
sql: SELECT etl_alias, state, count(*) AS runs, max(created_at) AS last_run, count(*) FILTER (WHERE created_at > now() - interval '8 days') AS runs_8d FROM <tabla_logs_etl> GROUP BY 1, 2 ORDER BY 1, 2

# C.4.4 — ¿cuánto tarda cada corrida? (fecha de fin registrada menos inicio)
mcp__postgres__query
sql: SELECT created_at, last_date_sync, created_at - last_date_sync AS duracion, state FROM <tabla_logs_etl> WHERE etl_alias = '<alias>' ORDER BY created_at DESC LIMIT 5

# C.4.5 — ¿cuántas filas por día entran al destino? (servidor OLAP)
mcp__postgres-olap__query
sql: SELECT created_at::date AS d, count(*) FROM <tabla_destino> WHERE created_at > now() - interval '5 days' GROUP BY 1 ORDER BY 1

# C.4.6 — ¿hay duplicados en el destino?
mcp__postgres-olap__query
sql: SELECT count(*) AS total, count(DISTINCT id) AS distinct_ids FROM <tabla_destino> WHERE created_at > now() - interval '3 days'

# C.4.7 — ¿cuántas filas de una tabla efímera ya no sirven?
mcp__postgres__query
sql: SELECT count(*) FILTER (WHERE expires_at < now()) AS expirados, count(*) AS total, min(created_at) FROM <tabla_tokens>

# C.4.8 — ¿el índice nuevo se está usando?
mcp__postgres__query
sql: SELECT indexrelname, idx_scan, idx_tup_read FROM pg_stat_user_indexes WHERE indexrelname IN ('<indice_nuevo_1>','<indice_nuevo_2>')

# C.4.9 — ¿cuántas entidades disparan un job por entidad? (explica calls/día)
mcp__postgres__query
sql: SELECT count(DISTINCT <columna>) FROM <tabla>
```

### C.5 Lo que el MCP no puede hacer (requiere psql o migración)

```sql
SELECT pg_stat_statements_reset();                     -- cerrar un ciclo de medición
ANALYZE <tabla>;  VACUUM (ANALYZE) <tabla>;            -- estadísticas y visibility map
CREATE INDEX CONCURRENTLY … ;  DROP INDEX CONCURRENTLY … ;
REINDEX INDEX CONCURRENTLY <indice>;
ALTER TABLE <tabla> SET (autovacuum_vacuum_scale_factor = 0.02);
VACUUM FULL <tabla>;                                   -- solo en ventana: toma lock exclusivo
```
