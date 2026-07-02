# Rollback / Savepoints

## Rollback

Deshace todos los cambios de la transacción actual, volviendo al estado previo al `BEGIN`.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- me doy cuenta que algo está mal
ROLLBACK; -- el UPDATE nunca sucedió
```

- Un `ROLLBACK` automático también ocurre si la conexión se cae o si una sentencia dentro de la transacción falla (según el motor: Postgres aborta toda la transacción ante cualquier error hasta el `ROLLBACK` explícito; MySQL suele ser más permisivo).

## Savepoints

Puntos de control **dentro** de una transacción que permiten deshacer solo una parte, sin perder todo lo hecho hasta ahí.

```sql
BEGIN;
INSERT INTO orders (id, user_id) VALUES (1, 10);

SAVEPOINT before_items;
INSERT INTO order_items (order_id, product_id) VALUES (1, 999); -- product_id inválido, falla

ROLLBACK TO SAVEPOINT before_items; -- deshace solo el INSERT de items, conserva el order

INSERT INTO order_items (order_id, product_id) VALUES (1, 5); -- reintento con dato correcto
COMMIT;
```

- `RELEASE SAVEPOINT before_items` — libera el savepoint sin deshacer nada (ya no se puede volver a él).
- Útil para simular "transacciones anidadas" (SQL no las soporta nativamente) o para manejar errores parciales en procesos batch sin perder todo el trabajo previo.

## Caso de uso típico: import masivo

```sql
BEGIN;
-- por cada fila del CSV:
SAVEPOINT row_start;
INSERT INTO products (...) VALUES (...);
-- si falla, ROLLBACK TO row_start y logueo el error, sigo con la próxima fila
-- si todo OK, RELEASE SAVEPOINT row_start
COMMIT; -- al final, todas las filas válidas quedan persistidas
```

Relacionado: [ACID / transacciones / isolation levels](acid-transacciones-isolation.md).
