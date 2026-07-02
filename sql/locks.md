# Locks (shared/exclusive/MVCC)

Mecanismos de control de concurrencia: cómo la base evita que transacciones simultáneas se pisen los datos.

## Pessimistic vs Optimistic locking

- **Pessimistic**: asumo que va a haber conflicto, así que bloqueo el recurso antes de tocarlo (`SELECT ... FOR UPDATE`). Otros procesos esperan.
- **Optimistic**: asumo que no va a haber conflicto, dejo que todos lean/escriban libremente, y verifico al momento de escribir (ej. columna `version`) si alguien más modificó el dato mientras tanto — si es así, rechazo el update.

```sql
-- Optimistic locking con columna version
UPDATE products SET stock = stock - 1, version = version + 1
WHERE id = 1 AND version = 5;
-- Si affected rows = 0 → alguien más lo modificó, hay que reintentar
```

## Shared lock (S) vs Exclusive lock (X)

| Lock | Permite a otros leer | Permite a otros escribir | Uso |
|---|---|---|---|
| **Shared (S)** | Sí (otro S) | No | `SELECT ... FOR SHARE` |
| **Exclusive (X)** | No | No | `UPDATE`, `DELETE`, `SELECT ... FOR UPDATE` |

```sql
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE; -- lock exclusivo de la fila
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT; -- libera el lock
```

## Row-level vs table-level

- La mayoría de los motores modernos (Postgres, MySQL InnoDB) lockean a **nivel fila** por defecto — mucho mejor para concurrencia que lockear la tabla entera.
- `LOCK TABLE` existe para casos puntuales (ej. migraciones, operaciones batch) pero bloquea a todos los demás.

## Deadlocks

Dos transacciones se bloquean mutuamente esperando el lock que tiene la otra.

```
T1: lock fila A, espera fila B
T2: lock fila B, espera fila A
→ deadlock
```

El motor detecta el ciclo y aborta una de las dos transacciones (la víctima recibe un error y debe reintentar). **Mitigación**: siempre lockear recursos en el mismo orden en toda la aplicación (ej. siempre por `id` ascendente).

## MVCC (Multi-Version Concurrency Control)

En vez de bloquear lecturas, Postgres y MySQL InnoDB mantienen **múltiples versiones** de cada fila:

- Cada transacción ve un *snapshot* consistente de los datos según su isolation level, sin bloquear a los que escriben.
- Un `UPDATE` no sobreescribe la fila en el lugar: crea una nueva versión y marca la vieja como obsoleta (Postgres) o la mueve al *undo log* (MySQL InnoDB).
- Un proceso de limpieza (`VACUUM` en Postgres) elimina versiones viejas que ya nadie necesita.

**Consecuencia práctica**: con MVCC, lectores nunca bloquean escritores ni viceversa (`SELECT` no espera a un `UPDATE` en curso) — solo escritor vs escritor genera contención real.

Ver también [ACID / transacciones / isolation levels](acid-transacciones-isolation.md), que depende directamente de estos mecanismos.
