# NoSQL

Bases de datos que se apartan del modelo relacional clásico (tablas + SQL + ACID estricto) para priorizar escalabilidad horizontal, esquemas flexibles o modelos de datos específicos.

## Categorías principales

| Tipo | Modelo | Ejemplos | Caso de uso típico |
|---|---|---|---|
| **Key-Value** | Clave → valor opaco | Redis, DynamoDB | Cache, sesiones, contadores, feature flags |
| **Documento** | Documentos JSON/BSON anidados | MongoDB, Couchbase | Datos semi-estructurados, schemas que cambian seguido |
| **Column-family** | Filas con columnas dinámicas agrupadas por familia | Cassandra, HBase | Escritura masiva, series temporales, alta disponibilidad multi-región |
| **Grafo** | Nodos + relaciones (edges) | Neo4j, Amazon Neptune | Datos altamente relacionales (redes sociales, recomendaciones, fraude) |

## Por qué NoSQL: el trade-off central

Relacional prioriza **consistencia e integridad** (constraints, foreign keys, transacciones ACID multi-tabla). NoSQL generalmente prioriza **disponibilidad y escalabilidad horizontal**, sacrificando algo de consistencia o de expresividad de queries.

## CAP Theorem

En un sistema distribuido, ante una partición de red (**P**, inevitable en la práctica), hay que elegir entre:

- **C**onsistency: todos los nodos ven el mismo dato al mismo tiempo.
- **A**vailability: el sistema sigue respondiendo aunque algunos nodos no puedan comunicarse entre sí.

| Sistema | Prioriza |
|---|---|
| Postgres/MySQL (single-node) | CA (no aplica P en un solo nodo) |
| MongoDB (config default) | CP |
| Cassandra, DynamoDB | AP (consistencia eventual) |

## Consistencia eventual

En sistemas AP (ej. Cassandra), un write puede no verse inmediatamente en todos los nodos — eventualmente converge, pero una lectura inmediata después de un write puede devolver el dato viejo. Aceptable para casos como contadores de likes; no aceptable para saldo de una cuenta bancaria.

## Documento — ejemplo (MongoDB)

```js
db.users.insertOne({
  name: "Ana",
  emails: ["ana@mail.com", "ana@work.com"], // array, sin necesidad de tabla aparte
  address: { city: "Buenos Aires", zip: "1000" } // anidado, sin JOIN
});
```

Ventaja: no hace falta definir el schema por adelantado, y datos anidados/relacionados van en un solo documento (menos joins). Desventaja: duplicación de datos, más difícil garantizar integridad referencial.

## Cuándo elegir NoSQL vs SQL

**NoSQL cuando:**
- El schema cambia frecuentemente o es muy heterogéneo entre registros.
- Necesitás escalar escritura horizontalmente a través de muchos nodos/regiones.
- El acceso a datos es simple (por key) y no necesitás joins complejos ni transacciones multi-entidad.

**SQL cuando:**
- Necesitás integridad referencial fuerte y transacciones ACID entre múltiples entidades (ej. sistemas financieros).
- Las relaciones entre entidades son el corazón del dominio y se consultan con queries complejas (joins, agregaciones).
- El equipo ya tiene expertise y tooling maduro en el ecosistema SQL.

En la práctica, muchos sistemas usan **ambos** (polyglot persistence): Postgres para el dominio transaccional, Redis para cache/sesiones, Elasticsearch para búsqueda full-text.

Relacionado: [Sharding vs partitioning](sharding-vs-partitioning.md), [ACID / transacciones / isolation levels](acid-transacciones-isolation.md).
