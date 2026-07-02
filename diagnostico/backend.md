# Diagnóstico Backend

Causas más comunes de lentitud del lado del servidor, de más a menos frecuentes.

## Problema N+1

Por cada elemento de una colección se dispara una query/request adicional, en vez de traer todo en una sola operación. Clásico en ORMs con lazy loading.

```js
// ❌ 1 query para pedidos + N queries, una por cada pedido, para su usuario
const orders = await Order.findAll();
for (const o of orders) { o.user = await User.findById(o.userId); }

// ✅ 1 sola query con join/eager loading
const orders = await Order.findAll({ include: User });
```

## Falta de cache

Recalcular o refetchear datos que cambian poco (config, catálogos, resultados de queries costosas) en cada request. Agregar cache en memoria (Redis) con invalidación adecuada — ver [system-design](../system-design/README.md).

```js
// ✅ Node + Redis: cache-aside pattern
async function getProduct(id) {
  const cached = await redis.get(`product:${id}`);
  if (cached) return JSON.parse(cached);

  const product = await db.products.findById(id);
  await redis.set(`product:${id}`, JSON.stringify(product), 'EX', 300); // TTL 5 min
  return product;
}
```

## Clustering

Un solo proceso Node/Python no aprovecha todos los cores de la máquina. Usar `cluster` module, PM2 en modo cluster, o múltiples workers/gunicorn para escalar verticalmente dentro del mismo servidor.

```js
// ✅ Node: un worker por core
import cluster from 'node:cluster';
import os from 'node:os';

if (cluster.isPrimary) {
  os.cpus().forEach(() => cluster.fork());
} else {
  startServer(); // cada worker corre su propia instancia del servidor
}
```

## Memoria

Memory leaks (listeners no removidos, caches sin límite, closures que retienen referencias) degradan performance con el tiempo y eventualmente fuerzan swapping o crashes por OOM. Tomar dos **heap snapshots** (ej. Chrome DevTools → Memory, o `node --inspect`) separados por un rato de tráfico normal y comparar: si un tipo de objeto crece entre snapshots sin bajar nunca, ese es el que se está leakeando — el snapshot te dice *qué* objeto retiene la memoria, no solo cuánta memoria hay.

```js
// ❌ leak: el listener nunca se remueve, se acumulan en cada request
function handleRequest() { emitter.on('data', onData); }

// ✅ remover al terminar
function handleRequest() {
  emitter.on('data', onData);
  emitter.once('end', () => emitter.off('data', onData));
}
```

## CPU bound

Operaciones intensivas en cómputo (compresión, criptografía, procesamiento de imágenes) bloquean el event loop en runtimes single-threaded (Node). Delegar a workers, colas de background jobs, o servicios separados.

```js
// ✅ Node: worker_threads saca el cómputo pesado del event loop principal
import { Worker } from 'node:worker_threads';
const worker = new Worker('./resize-image.worker.js', { workerData: { path } });
```

## HTTP chaining

Un endpoint que internamente hace múltiples llamadas HTTP secuenciales a otros servicios, sumando latencias en cascada en vez de paralelizarlas (`Promise.all`) o resolverlas con un solo request agregado.

```ts
// ❌ 300ms de latencia acumulada (100ms cada request, secuencial)
const user = await fetchUser(id);
const orders = await fetchOrders(id);
const reviews = await fetchReviews(id);

// ✅ 100ms total, en paralelo
const [user, orders, reviews] = await Promise.all([
  fetchUser(id), fetchOrders(id), fetchReviews(id),
]);
```

## Dependencias innecesarias

Llamar a un servicio externo, cargar una librería pesada, o hacer una query cuyo resultado no se usa. Cada dependencia agrega latencia y puntos de falla — auditar qué es realmente necesario en el hot path.

```py
# ❌ importa toda la librería para un solo cálculo
import pandas as pd
avg = pd.Series(values).mean()

# ✅ statistics (stdlib) alcanza, sin la dependencia pesada
from statistics import mean
avg = mean(values)
```

## Algoritmos no optimizados

Complejidad algorítmica alta (`O(n²)` donde alcanza `O(n log n)` o `O(n)`) sobre colecciones grandes. Antes de optimizar infraestructura, revisar si el algoritmo es el problema.

```py
# ❌ O(n²): busca en la lista por cada elemento
duplicates = [x for x in items if items.count(x) > 1]

# ✅ O(n): usa un set/dict para lookup O(1)
seen, duplicates = set(), set()
for x in items:
    (duplicates if x in seen else seen).add(x)
```

## No liberar recursos del sistema

File handles, sockets, o buffers que quedan abiertos sin cerrarse (falta de `finally`/`using`/context managers) agotan recursos del proceso con el tiempo.

```py
# ❌ si algo falla antes del close(), el file handle queda abierto
f = open('data.csv')
process(f)
f.close()

# ✅ context manager: se cierra siempre, incluso ante una excepción
with open('data.csv') as f:
    process(f)
```

## Paralelismo y concurrencia

Trabajo independiente resuelto secuencialmente cuando podría correr en paralelo (`await` uno por uno en vez de `Promise.all`), o al revés: paralelismo sin control que satura CPU/conexiones.

```py
# ✅ Python: correr tareas independientes en paralelo con asyncio
import asyncio
results = await asyncio.gather(fetch_a(), fetch_b(), fetch_c())
```

## Conexiones mal gestionadas

No usar **connection pooling** hacia la base de datos (abrir una conexión nueva por request es costoso), o no liberar conexiones al pool después de usarlas, agotando el pool bajo carga.

```js
// ✅ pool reutilizable, no una conexión nueva por request
const pool = new Pool({ max: 20 });
```

---
Para confirmar antes de optimizar: profiler del runtime (`node --prof`, `py-spy`), APM (New Relic/Datadog), y logs de latencia por endpoint.
