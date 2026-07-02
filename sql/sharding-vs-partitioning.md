# Sharding vs Partitioning

Ambas son estrategias para dividir datos y escalar más allá de lo que soporta un solo nodo/tabla, pero operan en niveles distintos.

## Partitioning

Dividir una tabla **grande en un mismo servidor/instancia** en sub-tablas más chicas (particiones), transparente para las queries.

### Tipos

| Tipo | Criterio | Ejemplo |
|---|---|---|
| **Range** | Rango de valores | Particionar `orders` por mes (`created_at`) |
| **List** | Valores discretos | Particionar `users` por país (`AR`, `BR`, `MX`) |
| **Hash** | Hash de una columna, distribución uniforme | Particionar por `hash(user_id) % 4` |

```sql
-- Postgres: partition by range
CREATE TABLE orders (
  id bigint, created_at date, total numeric
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2024 PARTITION OF orders
  FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

**Beneficios**: queries que filtran por la columna de partición solo tocan la partición relevante (*partition pruning*), maintenance más rápido (`VACUUM`, borrar una partición vieja entera en vez de `DELETE` fila por fila).

**Sigue siendo un solo servidor**: no resuelve límites de CPU/disco/memoria de una sola instancia.

## Sharding

Dividir los datos **entre múltiples servidores/instancias** independientes (cada shard es una base de datos separada, potencialmente en distinta máquina).

### Estrategias

- **Key-based (hash) sharding**: `shard = hash(user_id) % N`. Distribución uniforme, pero resharding (agregar un nodo) es costoso porque cambia el mapeo de casi todas las keys.
- **Range-based sharding**: shard según rango de la key (ej. usuarios A-M en shard 1, N-Z en shard 2). Fácil de razonar, pero puede generar *hotspots* si el tráfico no está uniformemente distribuido.
- **Directory-based sharding**: una tabla de lookup mapea cada key a su shard. Flexible (permite mover keys individuales) pero agrega un punto de indirección/fallo.

### Desafíos de sharding

- **Cross-shard queries**: un `JOIN` entre datos de distintos shards no es nativo — hay que resolverlo en la capa de aplicación o con un query router.
- **Transacciones distribuidas**: mantener ACID entre shards requiere protocolos como *two-phase commit* o aceptar consistencia eventual.
- **Resharding**: agregar/quitar nodos implica redistribuir datos, operación delicada en producción (mitigado con *consistent hashing*).
- **Claves foráneas** entre shards no se pueden garantizar a nivel motor.

## Tabla comparativa

| | Partitioning | Sharding |
|---|---|---|
| Nivel | Una tabla, un servidor | Múltiples servidores |
| Objetivo | Manejabilidad, maintenance, partition pruning | Escalar horizontalmente más allá de un solo servidor |
| Transparencia para queries | Alta (el motor lo resuelve) | Baja (requiere lógica de routing en la app o un proxy) |
| Resuelve límite de hardware de un nodo | No | Sí |

## En la práctica

Se combinan: cada shard puede a su vez estar particionado internamente. Ejemplo: 8 shards de Postgres, cada uno con la tabla `orders` particionada por mes.

Relacionado: [Normalización](normalizacion.md) (denormalización suele acompañar sharding para evitar cross-shard joins), [NoSQL](nosql.md) (muchas bases NoSQL shardean nativamente, ej. Cassandra, MongoDB).
