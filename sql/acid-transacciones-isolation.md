# ACID / Transacciones / Isolation Levels

## ACID

- **Atomicity**: la transacción se aplica completa o no se aplica nada (`BEGIN`/`COMMIT`/`ROLLBACK`).
- **Consistency**: la transacción lleva la base de un estado válido a otro estado válido (respeta constraints, foreign keys, triggers).
- **Isolation**: transacciones concurrentes no se pisan entre sí — cada una ve un estado coherente, controlado por el **isolation level**.
- **Durability**: una vez hecho `COMMIT`, el cambio sobrevive a un crash (persistido en disco / WAL).

## Transacciones básicas

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT; -- o ROLLBACK si algo falla
```

Si el proceso muere entre los dos `UPDATE` sin `COMMIT`, al reconectar la transacción no existe: ninguno de los dos cambios se aplicó (atomicidad).

## Los tres "phenomena" que definen los isolation levels

| Fenómeno | Qué pasa |
|---|---|
| **Dirty read** | Leo un dato que otra transacción escribió pero todavía no hizo commit (y puede hacer rollback). |
| **Non-repeatable read** | Leo la misma fila dos veces en la misma transacción y obtengo valores distintos porque otra transacción la modificó y comiteó en el medio. |
| **Phantom read** | Repito la misma query con un `WHERE` y aparecen/desaparecen filas porque otra transacción insertó/borró filas que matchean el filtro. |

## Isolation levels (SQL standard)

| Nivel | Dirty read | Non-repeatable read | Phantom read |
|---|---|---|---|
| Read Uncommitted | Posible | Posible | Posible |
| Read Committed | Evitado | Posible | Posible |
| Repeatable Read | Evitado | Evitado | Posible* |
| Serializable | Evitado | Evitado | Evitado |

\* En Postgres, `Repeatable Read` usa snapshot MVCC y en la práctica también evita phantom reads (más estricto que el estándar SQL).

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
-- ...
COMMIT;
```

## Default por motor

- **Postgres**: `Read Committed` por defecto.
- **MySQL (InnoDB)**: `Repeatable Read` por defecto.
- **SQL Server**: `Read Committed` por defecto.

A mayor isolation level → mayor seguridad, pero más contención (locks) y más riesgo de errores de tipo *serialization failure* que requieren retry de la transacción. Es un trade-off consistencia vs throughput.

Ver también [Locks (shared/exclusive/MVCC)](locks.md) — el mecanismo interno que hace cumplir estos niveles — y [Rollback / savepoints](rollback-savepoints.md).
