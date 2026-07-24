# Migraciones de Base de Datos

Cómo evoluciona el schema de una base de datos a lo largo del tiempo sin romper producción — el desafío no es escribir el `ALTER TABLE`, es coordinarlo con el código de la app que ya está corriendo.

## 1. Versionado de schema

Cada cambio de schema (agregar columna, crear tabla, cambiar un tipo) se captura en un archivo de migración versionado y ordenado (numeración o timestamp), aplicado siempre en secuencia. Una tabla especial (ej. `schema_migrations`) registra qué migraciones ya corrieron en cada entorno — sin ese registro no hay forma confiable de saber si el schema de producción está al día con el código, ni de aplicar exactamente lo que falta al levantar un entorno nuevo.

```sql
-- 001_create_users_table.sql
CREATE TABLE users (id SERIAL PRIMARY KEY, email TEXT NOT NULL);

-- 002_add_active_column.sql
ALTER TABLE users ADD COLUMN active BOOLEAN DEFAULT true;

-- la propia herramienta de migraciones guarda qué versiones ya se aplicaron
SELECT * FROM schema_migrations;
-- version | applied_at
-- 001     | 2026-01-10
-- 002     | 2026-02-03
```

## 2. Forward-only vs reversible

Una migración **reversible** define un paso `up` (aplicar) y un `down` (deshacer), pensado para poder volver atrás si algo sale mal. Una migración **forward-only** no tiene `down`: si algo sale mal, se escribe una migración nueva que corrige el problema, nunca se "retrocede".

```sql
-- 002_add_active_column.up.sql
ALTER TABLE users ADD COLUMN active BOOLEAN DEFAULT true;

-- 002_add_active_column.down.sql
ALTER TABLE users DROP COLUMN active;
-- ⚠️ si entre el up y el down alguien ya escribió datos reales en esa columna, se pierden al bajar
```

En la práctica, muchos equipos terminan yendo forward-only aunque la herramienta soporte `down`: el `down` casi nunca se prueba (nadie lo corre hasta el día que lo necesita desesperadamente), y para ese entonces los datos ya cambiaron de forma que el `down` original ya no es válido — revertir schema después de que se escribieron datos reales es inherentemente más arriesgado que revertir código de aplicación.

## 3. Cómo se coordina con deploys

El problema no es la migración en sí, es el **orden** respecto al deploy del código. Con más de una instancia de la app corriendo (deploy rolling, el caso normal en cualquier sistema con más de un servidor), durante la ventana del rollout coexisten código viejo y código nuevo al mismo tiempo. Una migración que no es compatible con el código viejo (ej. renombrar una columna) rompe las instancias viejas que todavía están sirviendo tráfico mientras el nuevo código termina de desplegarse.

```sql
-- ❌ un rename directo, deployado junto con el código nuevo:
-- durante los segundos/minutos en que el código viejo todavía corre,
-- cada request que toque esta columna falla porque "email" ya no existe
ALTER TABLE users RENAME COLUMN email TO email_address;
```

La solución no es "hacerlo más rápido" — es partir el cambio en pasos que sean compatibles con ambas versiones del código al mismo tiempo. Eso es el **expand/contract pattern**.

## 4. Expand/contract pattern (zero-downtime)

En vez de un cambio destructivo de una sola vez, se divide en fases donde el schema siempre es válido tanto para el código viejo como para el nuevo:

```sql
-- Fase 1 — Expand: agregar la columna nueva sin tocar la vieja.
-- El código viejo ni se entera de que existe, sigue funcionando igual.
ALTER TABLE users ADD COLUMN email_address TEXT;

-- Fase 2 — Backfill + dual write: copiar los datos existentes a la columna nueva,
-- mientras el código de la app (ya deployado) escribe en ambas columnas a la vez.
UPDATE users SET email_address = email WHERE email_address IS NULL;

-- Fase 3 — Deploy del código que lee/escribe solo email_address,
-- recién una vez confirmado que el backfill terminó y no quedan instancias viejas corriendo.

-- Fase 4 — Contract: borrar la columna vieja, en una migración separada y posterior,
-- solo después de confirmar que nada la sigue usando.
ALTER TABLE users DROP COLUMN email;
```

Cada fase, por separado, es compatible con el código viejo y con el nuevo — nunca hay un momento en el que una versión de la app rompa contra el schema actual. El costo es obvio: son 4 pasos (y 4 deploys/migraciones) en vez de uno, pero es el precio de no tener downtime.

## Nota sobre locks durante la migración

Un `ALTER TABLE` no es gratis en tablas grandes: agregar una columna con un valor default en versiones viejas de Postgres (<11) reescribía la tabla entera bajo un lock exclusivo, bloqueando lecturas y escrituras durante toda la operación. Ver [Locks](locks.md) — vale la pena confirmar qué tipo de lock toma cada operación de tu motor específico antes de correr una migración sobre una tabla con tráfico real en producción.

---
Relacionado: [Locks](locks.md), [ACID / transacciones / isolation levels](acid-transacciones-isolation.md), [Atributos de calidad de sistemas](../system-design/atributos-de-calidad.md) (Disponibilidad).
