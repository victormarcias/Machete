# Atributos de Calidad de Sistemas ("-ilities")

Propiedades no funcionales que definen qué tan bueno es un diseño más allá de "funciona" — la mayoría de las decisiones de arquitectura son tradeoffs conscientes entre estos atributos, no la maximización de uno solo.

## Escalabilidad

Capacidad de un sistema de manejar más carga (usuarios, requests, datos) sumando recursos, sin degradar performance ni requerir un rediseño. El bloqueo más común: guardar estado en memoria del proceso, lo que ata un usuario a una instancia específica e impide escalar horizontalmente sin problemas.

```js
// ❌ sesión en memoria del proceso: no escala horizontalmente —
// el siguiente request de ese usuario puede caer en otra instancia que no tiene esta sesión
const sessions = {};
app.post('/login', (req, res) => { sessions[req.body.userId] = { loggedIn: true }; });

// ✅ estado en un store externo compartido (Redis): cualquier instancia puede atenderlo
app.post('/login', async (req, res) => {
  await redis.set(`session:${req.body.userId}`, JSON.stringify({ loggedIn: true }));
});
```

Ver [Escalabilidad vertical vs horizontal](../diagnostico/devops.md) para la implementación concreta a nivel infraestructura, y [Sharding vs partitioning](../sql/sharding-vs-partitioning.md) para el ángulo de base de datos.

## Inmutabilidad

Una vez creado un dato, no cambia — cualquier "modificación" produce una copia nueva. Importa porque elimina una clase entera de bugs de concurrencia (nadie puede mutar algo que otro está leyendo al mismo tiempo — ver [Locks](../sql/locks.md)), hace el estado predecible, y permite detectar cambios comparando referencias en vez de hacer una comparación profunda costosa — es la base de cómo React decide si re-renderizar.

```python
# ❌ mutable: cualquier código con una referencia al objeto puede alterarlo
# sin que el resto de la app se entere, generando efectos secundarios ocultos
config = {"retries": 3}
def process(cfg):
    cfg["retries"] = 0

# ✅ inmutable: cualquier cambio genera un objeto nuevo, el original queda intacto
from dataclasses import dataclass, replace

@dataclass(frozen=True)
class Config:
    retries: int = 3

new_config = replace(config, retries=0)  # config original no cambió
```

```ts
// ❌ mutar el estado directo: React compara por referencia y no detecta el cambio (mismo objeto)
const [user, setUser] = useState({ name: 'Ana', age: 30 });
user.age = 31; // no dispara re-render

// ✅ crear un objeto nuevo: la referencia cambia, React sí re-renderiza
setUser({ ...user, age: 31 });
```

## Disponibilidad

Proporción del tiempo que el sistema responde correctamente. Se mide en "nueves" (99.9% ≈ 8.7 horas de downtime al año, 99.99% ≈ 52 minutos). Se logra con redundancia: más de una instancia corriendo, en más de una zona de disponibilidad, con un load balancer que deja de mandar tráfico a la que falla.

```yaml
# ✅ K8s: sin esto, un pod caído sigue recibiendo tráfico hasta que alguien lo note
readinessProbe:
  httpGet: { path: /health, port: 3000 }
  periodSeconds: 10
```

Trade-off directo con **Consistencia** (ver más abajo) — es la esencia del CAP theorem, ver [NoSQL](../sql/nosql.md).

## Consistencia

Todos los nodos/lectores del sistema ven el mismo dato al mismo tiempo. **Consistencia fuerte**: una escritura es visible inmediatamente para toda lectura posterior (más simple de razonar, más caro de lograr en sistemas distribuidos). **Consistencia eventual**: una escritura tarda un rato en propagarse a todos los nodos, pero converge (más disponible, más barato, pero un usuario puede ver datos viejos por un momento).

```sql
-- ✅ Postgres, consistencia fuerte dentro de una transacción:
-- ninguna otra conexión ve el balance actualizado hasta el COMMIT
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

Ver [ACID / isolation levels](../sql/acid-transacciones-isolation.md) para consistencia fuerte, y [NoSQL](../sql/nosql.md) para el trade-off con disponibilidad (CAP theorem).

## Tolerancia a fallos

El sistema sigue funcionando (aunque sea en modo degradado) cuando falla un componente, en vez de caerse en cascada. El patrón más común: **circuit breaker** — deja de intentar contra un servicio que ya demostró estar caído, en vez de acumular timeouts que agotan recursos propios.

```ts
// ✅ circuit breaker simplificado: corta los intentos tras fallos repetidos
class CircuitBreaker {
  private failures = 0;
  private open = false;

  async call<T>(fn: () => Promise<T>): Promise<T> {
    if (this.open) throw new Error('Circuit open — servicio no disponible, no reintentamos');
    try {
      const result = await fn();
      this.failures = 0; // se recupera tras un éxito
      return result;
    } catch (err) {
      if (++this.failures >= 3) this.open = true; // corta tras 3 fallos seguidos
      throw err;
    }
  }
}
```

## Idempotencia

Ejecutar la misma operación varias veces produce el mismo resultado que ejecutarla una sola vez. Crítico para reintentos seguros ante fallos de red: si un cliente manda un request, se cae la conexión antes de recibir la respuesta, y reintenta — ¿el servidor ya procesó el primer intento o no? Sin idempotencia, ese reintento puede duplicar un cobro.

```python
# ❌ no idempotente: reintentar tras un timeout puede cobrar dos veces
def charge(amount, card):
    return payment_gateway.create_charge(amount, card)

# ✅ idempotente: la misma idempotency_key nunca genera un segundo cobro,
# el servidor la usa para detectar "ya procesé este request antes"
def charge(amount, card, idempotency_key):
    return payment_gateway.create_charge(amount, card, idempotency_key=idempotency_key)
```

En HTTP, `GET`/`PUT`/`DELETE` están definidos como idempotentes por spec; `POST` no — por eso `POST` es el verbo más riesgoso para reintentar automáticamente sin una idempotency key.

## Observabilidad

Qué tan bien se puede entender el estado interno de un sistema mirando sus outputs externos (logs, métricas, traces), sin tener que modificarlo o debuggear en vivo. El pilar que más se olvida: sin un **correlation ID** que viaje entre servicios, un log individual no sirve para reconstruir qué pasó con un request que cruzó 5 microservicios.

```js
// ✅ log estructurado con correlation id — permite rastrear un request across servicios
logger.info({ event: 'order_created', orderId, correlationId: req.headers['x-correlation-id'] });
```

## Elasticidad

Caso particular de escalabilidad: la capacidad de escalar automáticamente hacia arriba **y hacia abajo** según la demanda en tiempo real, sin intervención manual. La diferencia con "escalabilidad" a secas es esa automatización — un sistema puede ser escalable (soporta más carga si le agregás recursos a mano) sin ser elástico (nadie se los agrega solo).

```yaml
# ✅ K8s HPA: agrega/quita pods solo, según el uso real de CPU
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
```

## Resumen

| Atributo | En una frase |
|---|---|
| Escalabilidad | Aguanta más carga sumando recursos |
| Inmutabilidad | Los datos no cambian, se reemplazan |
| Disponibilidad | El sistema responde cuando lo necesitás |
| Consistencia | Todos ven el mismo dato al mismo tiempo |
| Tolerancia a fallos | Sigue funcionando aunque algo se rompa |
| Idempotencia | Repetir la operación no cambia el resultado |
| Observabilidad | Podés entender qué pasa adentro desde afuera |
| Elasticidad | Escala sola, arriba y abajo, según demanda |

Estos atributos suelen tironear entre sí (el ejemplo clásico: más Consistencia generalmente cuesta Disponibilidad). Toda decisión de arquitectura es elegir conscientemente qué priorizar para el caso de uso — no existe un diseño que maximice todos a la vez.
