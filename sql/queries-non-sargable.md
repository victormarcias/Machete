# Queries non-sargable

**SARGable** = "Search ARGument ABLE": una condición que el motor puede resolver usando un índice directamente. Una query **non-sargable** obliga a un *full scan* aunque exista un índice sobre la columna, porque la condición no permite usarlo.

## Patrones comunes que rompen el índice

### 1. Función sobre la columna indexada

```sql
❌ SELECT * FROM users WHERE UPPER(email) = 'ANA@MAIL.COM';
✅ SELECT * FROM users WHERE email = 'ana@mail.com'; -- normalizar el dato al guardar, no al leer
✅ CREATE INDEX idx_email_upper ON users(UPPER(email)); -- o índice funcional, si no se puede evitar
```

### 2. Wildcard al principio de un LIKE

```sql
❌ SELECT * FROM users WHERE name LIKE '%ana%';  -- no puede usar B-tree, busca en todos lados
✅ SELECT * FROM users WHERE name LIKE 'ana%';   -- prefijo conocido, sí usa el índice
   -- Para búsqueda de substring real, usar full-text search (GIN/tsvector) o trigram (pg_trgm)
```

### 3. Conversión implícita de tipo

```sql
❌ SELECT * FROM orders WHERE order_id = '123';  -- order_id es int, compara con string, fuerza cast
✅ SELECT * FROM orders WHERE order_id = 123;
```

### 4. Operación aritmética sobre la columna

```sql
❌ SELECT * FROM sales WHERE price * 1.21 > 100;
✅ SELECT * FROM sales WHERE price > 100 / 1.21;  -- mover el cálculo al literal, no a la columna
```

### 5. `OR` en columnas distintas

```sql
❌ SELECT * FROM users WHERE email = 'x@mail.com' OR phone = '123456';
   -- muchas veces el planner no puede combinar dos índices distintos eficientemente
✅ -- Evaluar UNION de dos queries indexadas por separado:
   SELECT * FROM users WHERE email = 'x@mail.com'
   UNION
   SELECT * FROM users WHERE phone = '123456';
```

### 6. `NOT IN`, `!=`, `<>`

```sql
❌ SELECT * FROM orders WHERE status != 'cancelled';
   -- baja selectividad + negación, generalmente termina en seq scan
```

### 7. `OR IS NULL` combinado con otras condiciones, y funciones de fecha

```sql
❌ SELECT * FROM orders WHERE EXTRACT(YEAR FROM created_at) = 2024;
✅ SELECT * FROM orders WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
```

## Regla general

Si la columna indexada aparece **dentro** de una función, cast, o cálculo del lado izquierdo de la comparación, el motor generalmente no puede usar el índice — hay que dejar la columna "pelada" y mover la transformación al literal del otro lado.

## Cómo detectarlo

```sql
EXPLAIN ANALYZE SELECT ...;
```

Buscar `Seq Scan` donde se esperaría `Index Scan`. Ver [db-diagnostico](../db-diagnostico/README.md).

Relacionado: [Índices](indices.md).
