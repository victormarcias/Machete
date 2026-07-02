# Normalización (1NF-5NF)

Proceso de organizar columnas y tablas para reducir redundancia y evitar anomalías de inserción, actualización y borrado.

## Las tres anomalías que la normalización evita

- **Insert anomaly**: no podés agregar un dato porque depende de otro que todavía no existe (ej. no podés agregar un curso sin un alumno inscripto, si alumno y curso están en la misma tabla).
- **Update anomaly**: el mismo dato repetido en varias filas — si cambia, hay que actualizar todas las filas o quedan inconsistentes.
- **Delete anomaly**: borrar una fila borra información que no debería perderse (ej. borrar el único alumno de un curso borra el curso también).

## 1NF — Primera forma normal

- Cada columna tiene valores atómicos (no listas, no arrays, no CSV en un campo).
- No hay grupos repetidos (columnas `telefono1`, `telefono2`, `telefono3`).

```
❌ users(id, name, phones="123,456,789")
✅ users(id, name)  +  phones(id, user_id, phone)
```

## 2NF — Segunda forma normal

- Cumple 1NF **y** todo atributo no-clave depende de la **clave primaria completa** (relevante solo con claves compuestas).

```
❌ order_items(order_id, product_id, product_name, quantity)
   -- product_name depende solo de product_id, no de la clave completa (order_id, product_id)
✅ order_items(order_id, product_id, quantity)  +  products(product_id, product_name)
```

## 3NF — Tercera forma normal

- Cumple 2NF **y** no hay dependencias transitivas (un atributo no-clave que depende de otro atributo no-clave, en vez de depender directamente de la clave).

```
❌ employees(id, name, dept_id, dept_name)
   -- dept_name depende de dept_id, no directamente de id
✅ employees(id, name, dept_id)  +  departments(dept_id, dept_name)
```

## BCNF — Boyce-Codd (3.5NF)

- Versión más estricta de 3NF: toda dependencia funcional `X → Y` requiere que `X` sea una superclave. Resuelve casos raros donde 3NF todavía permite anomalías con claves candidatas superpuestas.

## 4NF — Cuarta forma normal

- Cumple BCNF **y** no hay dependencias multivaluadas independientes en la misma tabla.

```
❌ employee_skills_languages(emp_id, skill, language)
   -- skills y languages son independientes entre sí, mezclarlos genera filas redundantes (producto cartesiano)
✅ employee_skills(emp_id, skill)  +  employee_languages(emp_id, language)
```

## 5NF — Quinta forma normal (Project-Join Normal Form)

- Cumple 4NF **y** la tabla no se puede descomponer en tablas más chicas sin perder información al volver a unirlas (join lossless). Aplica a casos de relaciones ternarias complejas — poco común en diseño práctico del día a día.

## Tabla resumen

| Forma | Requisito clave |
|---|---|
| 1NF | Valores atómicos, sin grupos repetidos |
| 2NF | 1NF + sin dependencia parcial de clave compuesta |
| 3NF | 2NF + sin dependencias transitivas |
| BCNF | 3NF + toda determinante es superclave |
| 4NF | BCNF + sin dependencias multivaluadas |
| 5NF | 4NF + sin dependencias de join |

## Denormalización: el trade-off

En la práctica, 3NF suele ser el punto de equilibrio. Denormalizar (duplicar datos a propósito) es válido para:

- Optimizar lecturas en sistemas con mucho más tráfico de lectura que escritura (ej. guardar `product_name` en `order_items` como snapshot histórico del precio/nombre al momento de la compra).
- Evitar joins costosos en queries de alta frecuencia (trade-off clásico: **consistencia vs performance**).

Ver también [Sharding vs partitioning](sharding-vs-partitioning.md) para estrategias a mayor escala.
