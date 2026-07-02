# Índices

Estructura de datos auxiliar que permite encontrar filas sin recorrer toda la tabla (evita el *full table scan*).

## Por qué existen

Sin índice, `WHERE email = 'x@mail.com'` es O(n): recorre todas las filas. Con un índice tipo B-tree sobre `email`, la búsqueda es O(log n).

```sql
CREATE INDEX idx_users_email ON users(email);
```

## Tipos más comunes

| Tipo | Uso típico | Motor |
|---|---|---|
| **B-tree** | Default. Igualdad, rangos (`<`, `>`, `BETWEEN`), `ORDER BY` | Todos |
| **Hash** | Solo igualdad (`=`), no sirve para rangos | Postgres, MySQL |
| **GIN** | Full-text search, arrays, JSONB | Postgres |
| **GiST** | Datos geométricos, rangos, búsquedas de proximidad | Postgres |
| **Bitmap** | Columnas de baja cardinalidad (pocos valores distintos) combinadas | Oracle, data warehouses |

## Índices compuestos

```sql
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);
```

- El orden de las columnas importa: este índice sirve para `WHERE user_id = ?` y para `WHERE user_id = ? AND created_at > ?`, pero **no** sirve eficientemente para `WHERE created_at > ?` solo (regla del *leftmost prefix*).

## Covering index

Un índice que incluye todas las columnas que la query necesita, permitiendo responder sin ir a la tabla (*index-only scan*):

```sql
CREATE INDEX idx_orders_covering ON orders(user_id) INCLUDE (status, total);
```

## Clustered vs non-clustered

- **Clustered**: los datos de la tabla se almacenan físicamente en el orden del índice (ej. la PK en SQL Server/MySQL InnoDB). Solo puede haber uno por tabla.
- **Non-clustered**: estructura separada que apunta a la ubicación de la fila. Puede haber varios.
- Postgres no tiene clustered index real (usa *heap* + índices que apuntan al heap; `CLUSTER` reordena físicamente una sola vez, no lo mantiene).

## Costo: no son gratis

- Cada índice acelera lecturas pero **ralentiza escrituras** (`INSERT`/`UPDATE`/`DELETE` deben mantener también el índice actualizado).
- Ocupan espacio en disco.
- Regla práctica: indexar columnas usadas en `WHERE`, `JOIN`, `ORDER BY` con alta selectividad (muchos valores distintos); evitar indexar columnas de baja cardinalidad (ej. `boolean`) salvo con índices parciales.

## Índice parcial (Postgres)

```sql
CREATE INDEX idx_active_users ON users(email) WHERE active = true;
```

Útil cuando solo un subconjunto de filas se consulta frecuentemente.

## Cómo saber si se está usando

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'x@mail.com';
```

Buscar `Index Scan` / `Index Only Scan` en el plan (vs `Seq Scan`). Ver [db-diagnostico](../db-diagnostico/README.md) para el detalle de `EXPLAIN ANALYZE`.

Relacionado: [Queries non-sargable](queries-non-sargable.md) — patrones de queries que impiden que el motor use el índice aunque exista.
