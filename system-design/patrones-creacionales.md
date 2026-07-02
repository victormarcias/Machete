# Patrones de Diseño — Creacionales

Patrones que abstraen la lógica de creación de objetos, para no acoplar el código cliente a clases concretas específicas.

## Factory Method

Define una forma de crear un objeto, pero deja que una función (o subclase) decida qué clase concreta instanciar. Útil cuando el tipo exacto a crear depende de algo que solo se sabe en runtime (config, tipo de dato, entorno).

```python
class MySQLConnection: ...
class PostgresConnection: ...

def create_connection(driver: str):
    if driver == "mysql":
        return MySQLConnection()
    elif driver == "postgres":
        return PostgresConnection()
    raise ValueError(f"Driver desconocido: {driver}")

conn = create_connection(config.db_driver)  # el resto del código no conoce la clase concreta
```

```ts
interface Connection { query(sql: string): Promise<unknown>; }
class MySQLConnection implements Connection { async query(sql: string) { /* ... */ } }
class PostgresConnection implements Connection { async query(sql: string) { /* ... */ } }

function createConnection(driver: 'mysql' | 'postgres'): Connection {
  return driver === 'mysql' ? new MySQLConnection() : new PostgresConnection();
}
```

## Abstract Factory

Un nivel arriba del Factory Method: una factory que crea familias enteras de objetos relacionados, garantizando que sean compatibles entre sí (ej. no mezclar por error el storage de un proveedor cloud con la cola de otro).

```ts
interface CloudFactory {
  createStorage(): Storage;
  createQueue(): Queue;
}

class AWSFactory implements CloudFactory {
  createStorage() { return new S3Storage(); }
  createQueue() { return new SQSQueue(); }
}

class GCPFactory implements CloudFactory {
  createStorage() { return new GCSStorage(); }
  createQueue() { return new PubSubQueue(); }
}

// setupInfra nunca puede terminar mezclando S3 con PubSub — la factory garantiza la familia correcta
function setupInfra(factory: CloudFactory) {
  const storage = factory.createStorage();
  const queue = factory.createQueue();
}
```

## Builder

Construir un objeto complejo paso a paso, separando la construcción de la representación final. Útil cuando el constructor directo tendría demasiados parámetros opcionales y se vuelve ilegible en el call site.

```python
# ❌ constructor con muchos parámetros opcionales, difícil de leer en el call site
query = Query(table="orders", where="status='paid'", order_by="created_at", limit=10, select=["id", "total"])

# ✅ Builder: cada método agrega una pieza, se lee casi como una oración
query = (
    QueryBuilder("orders")
    .select("id", "total")
    .where("status='paid'")
    .order_by("created_at")
    .limit(10)
    .build()
)
```

```ts
class QueryBuilder {
  private parts: string[] = [];
  constructor(private table: string) {}
  select(...cols: string[]) { this.parts.push(`SELECT ${cols.join(', ')} FROM ${this.table}`); return this; }
  where(cond: string) { this.parts.push(`WHERE ${cond}`); return this; }
  limit(n: number) { this.parts.push(`LIMIT ${n}`); return this; }
  build() { return this.parts.join(' '); }
}

const sql = new QueryBuilder('orders').select('id', 'total').where("status='paid'").limit(10).build();
```

## Singleton

Garantiza que una clase tenga una única instancia compartida en toda la app, con un punto de acceso global (ej. pool de conexión a DB, logger, config cargada una sola vez).

```python
class Logger:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

logger1, logger2 = Logger(), Logger()
assert logger1 is logger2  # misma instancia
```

```ts
class Logger {
  private static instance: Logger;
  private constructor() {}
  static getInstance(): Logger {
    if (!Logger.instance) Logger.instance = new Logger();
    return Logger.instance;
  }
}

const logger = Logger.getInstance();
```

**Cuidado**: es el patrón más criticado de la lista — introduce estado global compartido, dificulta el testing (no podés inyectar un mock fácilmente en su lugar) y esconde una dependencia que debería ser explícita en la firma de la clase que lo usa. En la práctica, muchos frameworks resuelven el mismo problema con **Dependency Injection** en vez de un Singleton manual (ver [DIP en SOLID](solid.md)).

---
Relacionado: [SOLID principles](solid.md) · catálogo completo en [refactoring.guru](https://refactoring.guru/design-patterns/creational-patterns).
