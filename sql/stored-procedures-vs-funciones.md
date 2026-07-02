# Stored Procedures vs Funciones

Ambos son código SQL/procedural almacenado y ejecutado del lado del servidor de base de datos, pero tienen diferencias clave.

## Diferencias principales

| | Stored Procedure | Función (Function) |
|---|---|---|
| Se invoca con | `CALL proc(...)` | `SELECT func(...)` — dentro de una query |
| Retorno | Opcional, puede devolver múltiples result sets o ninguno | Obligatorio, devuelve un valor (escalar, tabla, etc.) |
| Control transaccional | Puede hacer `COMMIT`/`ROLLBACK` propio (según motor) | No puede controlar transacciones — corre dentro de la transacción del caller |
| Uso en SELECT | No se puede usar dentro de un `SELECT`/`WHERE` | Sí, se puede usar en `SELECT`, `WHERE`, `JOIN` |
| Side effects | Pensado para tener efectos secundarios (INSERT/UPDATE/DELETE, lógica de negocio) | Debería ser preferentemente pura/determinística, aunque algunos motores permiten side effects |

## Ejemplo — Función (Postgres)

```sql
CREATE FUNCTION total_pedido(pedido_id int) RETURNS numeric AS $$
  SELECT SUM(price * quantity) FROM order_items WHERE order_id = pedido_id;
$$ LANGUAGE sql;

SELECT id, total_pedido(id) FROM orders; -- usable dentro de un SELECT
```

## Ejemplo — Stored Procedure (Postgres)

```sql
CREATE PROCEDURE archivar_pedidos_viejos() AS $$
BEGIN
  INSERT INTO orders_archive SELECT * FROM orders WHERE created_at < now() - interval '2 years';
  DELETE FROM orders WHERE created_at < now() - interval '2 years';
  COMMIT; -- un procedure puede manejar su propia transacción
END;
$$ LANGUAGE plpgsql;

CALL archivar_pedidos_viejos();
```

## Cuándo usar cada uno

- **Función**: cálculos reutilizables dentro de queries (ej. calcular un total, validar un formato, transformar un dato).
- **Procedure**: procesos batch, tareas administrativas, lógica que involucra múltiples pasos con su propio control transaccional (ETL, jobs programados).

## Trade-off: lógica en la DB vs en la aplicación

**A favor de stored procedures/funciones:**
- Menos round-trips de red (la lógica corre donde están los datos).
- Útil cuando varios servicios/lenguajes necesitan la misma lógica de forma consistente.

**En contra:**
- Difícil de versionar, testear y debuggear comparado con código de aplicación (menos tooling, sin type-checking del lenguaje principal).
- Acopla la lógica de negocio al motor de base de datos específico, dificultando migraciones.
- La tendencia moderna (microservicios, arquitecturas cloud-native) prefiere mantener la lógica de negocio en la capa de aplicación y usar la DB solo para persistencia — salvo casos puntuales de performance o integridad crítica.
