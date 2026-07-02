# Diagnóstico Base de Datos

Causas más comunes de lentitud a nivel base de datos, de más a menos frecuentes.

## `WHERE` mal usados

Condiciones que no permiten usar índices (funciones sobre la columna, wildcard al inicio de un `LIKE`, casts implícitos). Ver [Queries non-sargable](../sql/queries-non-sargable.md).

## Bloqueos y contenciones (transacciones)

Transacciones largas o mal aisladas retienen locks y hacen esperar a otras — deadlocks, transacciones que hacen `SELECT` innecesarios dentro del lock, isolation level más estricto de lo necesario. Ver [Locks](../sql/locks.md) y [ACID / isolation levels](../sql/acid-transacciones-isolation.md).

## Queries complejas (joins de más, `WHERE` con `LIKE`)

Joins que traen columnas/tablas que no se usan, o filtros tipo `LIKE '%texto%'` que fuerzan full scan en vez de aprovechar full-text search. Simplificar la query a lo que realmente se necesita.

## Falta de mantenimiento (reindexado y estadísticas)

Índices fragmentados y estadísticas desactualizadas hacen que el planner tome malas decisiones. `ANALYZE` recalcula la distribución de valores por columna que el planner usa para estimar cuántas filas va a devolver cada filtro — sin eso, puede subestimar el costo de un `Seq Scan` y descartar un índice que sí convenía usar. `VACUUM` recupera el espacio de las filas "muertas" que deja MVCC tras updates/deletes (ver [Locks](../sql/locks.md)); sin él, la tabla y sus índices se inflan y cada scan lee más páginas de las necesarias. Reindexar hace falta cuando el índice se fragmenta por mucha escritura/borrado y ya no queda balanceado. Correr `VACUUM ANALYZE` (Postgres) / `ANALYZE TABLE` (MySQL) periódicamente en vez de esperar a que el problema aparezca.

## Índices

Faltantes, redundantes, o mal diseñados (orden de columnas en compuestos, baja selectividad). Ver [Índices](../sql/indices.md).

## Execution plan

`EXPLAIN ANALYZE` para ver si el planner usa `Seq Scan` donde debería usar `Index Scan`, y dónde está el costo real (no siempre es donde uno asume).

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 42;
```

## Estadísticas del planner

El optimizador decide el plan según estadísticas de distribución de datos (cardinalidad, histogramas). Si están desactualizadas, elige planes subóptimos aunque el índice exista.

## Vistas materializadas

Precalculan y persisten el resultado de una query costosa (agregaciones, joins pesados), a costa de tener que refrescarlas. Útiles para dashboards/reportes que no necesitan datos al segundo.

```sql
CREATE MATERIALIZED VIEW ventas_por_mes AS
SELECT date_trunc('month', created_at) AS mes, SUM(total) FROM orders GROUP BY 1;

REFRESH MATERIALIZED VIEW ventas_por_mes;
```

## Cubos OLAP

Para analítica multidimensional pesada (BI, reportes históricos), separar el workload analítico (OLAP) del transaccional (OLTP) evita que queries de reporting compitan por recursos con el tráfico de producción. Ver [system-design](../system-design/README.md).

## Sharding entre distintas DBs (escalabilidad horizontal)

Cuando ya se indexó, cacheó y particionó bien, pero un solo servidor sigue sin dar abasto en CPU, memoria o IOPS, la siguiente escala es distribuir los datos entre múltiples instancias independientes (shards). El costo: joins y transacciones que antes eran nativos ahora cruzan servidores distintos, y hay que resolverlos a mano en la capa de aplicación. Ver [Sharding vs partitioning](../sql/sharding-vs-partitioning.md).

## Range partitioning

Caso particular de partitioning donde el criterio es un rango de valores, típicamente fechas (`orders_2024`, `orders_2025`). Sirve puntualmente cuando las queries casi siempre filtran por ese rango (ej. "pedidos del último mes"): el planner descarta directamente las particiones que no aplican (*partition pruning*) en vez de escanear la tabla completa, y permite borrar datos viejos eliminando una partición entera en vez de un `DELETE` masivo fila por fila. Ver [Sharding vs partitioning](../sql/sharding-vs-partitioning.md).

## Hardware específico

Antes de asumir que el problema es la query o el índice, descartar el piso físico: discos lentos (HDD vs SSD/NVMe) impactan directo en I/O de lecturas/escrituras, poca RAM limita cuánto del buffer pool/cache vive en memoria (más *cache misses* → más disco), y CPU insuficiente cuella queries con mucho cómputo (agregaciones, ordenamientos grandes). Un `EXPLAIN ANALYZE` con buen plan pero tiempos altos suele apuntar para acá.

## NoSQL (si hace falta)

Si después de indexar, cachear y particionar bien la base relacional sigue sin dar abasto, y el patrón de acceso es simple (búsquedas por key, sin joins complejos ni necesidad de integridad transaccional fuerte), vale la pena evaluar mover esa porción puntual del dominio a NoSQL en vez de seguir forzando el modelo relacional. No es un reemplazo general — es una herramienta para el caso donde el cuello de botella es escala horizontal, no relaciones. Ver [NoSQL](../sql/nosql.md).
