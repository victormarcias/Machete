# Sync vs Async en FastAPI

FastAPI acepta tanto `def` como `async def` en una ruta — la elección no es estética, tiene consecuencias reales de performance si se hace mal. Esto no es teoría de `endpoints-microservicios.md`; es específicamente cuándo async ayuda, cuándo no hace nada, y qué se rompe al migrar una app entera de sync a async.

## 1. Qué es el event loop y por qué importa

`asyncio` corre en **una sola hebra** con multitasking cooperativo: una corrutina cede el control (con `await`) en cada punto de espera, y el event loop aprovecha esa pausa para atender otras corrutinas. Si una corrutina nunca cede (hace trabajo bloqueante sin `await`), **frena el loop entero** — nada más corre hasta que termina, ni siquiera requests de otros usuarios.

```python
import asyncio, time

async def bad_task():
    time.sleep(5)  # ❌ bloquea el event loop entero — nada más corre durante estos 5 segundos

async def good_task():
    await asyncio.sleep(5)  # ✅ cede el control, el loop atiende otras corrutinas mientras tanto
```

## 2. `async def` vs `def` en una ruta — qué hace FastAPI distinto

Una ruta `async def` corre directo en el event loop, en la misma hebra que todo lo demás. Una ruta `def` normal, FastAPI la manda automáticamente a un **threadpool aparte** — por eso código sync bloqueante adentro no traba el resto de la app, a costa de usar una hebra del pool en vez del loop principal.

```python
# ✅ def normal: FastAPI la corre en threadpool automáticamente,
# el código bloqueante de acá adentro no frena el event loop principal
@app.get("/legacy")
def get_legacy_data():
    return sync_blocking_call()

# ✅ async def: corre en el event loop compartido — solo tiene sentido
# si TODO lo que hace adentro es realmente async (con await)
@app.get("/modern")
async def get_modern_data():
    return await async_non_blocking_call()
```

## 3. I/O-bound vs CPU-bound — la regla real

Async brilla en trabajo **I/O-bound** (llamada de red a otro servicio, query a la DB, leer un archivo): la corrutina pasa la mayor parte del tiempo esperando, y durante esa espera el event loop atiende otras requests. En trabajo **CPU-bound** (parsear un JSON gigante, resize de imágenes, un loop pesado) no hay "espera" que ceder — una sola hebra haciendo cómputo bloquea igual, tenga `async`/`await` o no. Ver [Diagnóstico Backend](../diagnostico/backend.md) (CPU bound) para cómo sacar ese trabajo del proceso principal.

## 4. Cuándo async no sirve (o empeora)

La trampa más común: declarar `async def` pero llamar adentro a una función sync bloqueante **sin** `await`. Eso no libera nada — bloquea el event loop para **todas** las requests concurrentes, algo peor que si esa misma ruta hubiera sido `def` a secas (que FastAPI habría mandado a un threadpool en vez de trabar el loop principal).

```python
# ❌ el peor caso: async def que llama a código sync bloqueante sin await —
# frena el event loop para TODAS las requests, no solo para esta
@app.get("/orders")
async def get_orders():
    return requests.get("http://otro-servicio/orders")  # librería sync, sin await

# ✅ si el trabajo interno es sync, usar def a secas — FastAPI lo manda a threadpool solo
@app.get("/orders")
def get_orders():
    return requests.get("http://otro-servicio/orders")

# ✅ o usar la versión async de la librería HTTP
@app.get("/orders")
async def get_orders():
    async with httpx.AsyncClient() as client:
        return await client.get("http://otro-servicio/orders")
```

## 5. SQLAlchemy async: engine, `AsyncSession`, `aiosqlite`

Para que las queries a la DB sean realmente no-bloqueantes hace falta un **driver async** (`aiosqlite` para SQLite, `asyncpg` para Postgres) y `create_async_engine` + `AsyncSession` — no alcanza con envolver la `Session` sync de siempre en un `async def`, eso seguiría bloqueando el loop en cada query.

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker

engine = create_async_engine("sqlite+aiosqlite:///./app.db")  # driver async, no el sync de siempre
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_db():
    async with AsyncSessionLocal() as session:
        yield session

@app.get("/orders/{order_id}")
async def get_order(order_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(Order).where(Order.id == order_id))
    return result.scalar_one_or_none()
```

## 6. Eager loading de relaciones en modo async

El lazy loading clásico (acceder a `order.items` recién cuando se lee el atributo, disparando una query nueva en ese momento) directamente **falla** en SQLAlchemy async con un error `MissingGreenlet` — cargar la relación ahí requeriría hacer I/O de forma síncrona fuera de un contexto `await`, y el driver async no lo permite. La solución es cargar la relación por adelantado con `selectinload`/`joinedload` en la query original.

```python
from sqlalchemy.orm import selectinload

# ❌ lazy loading: acceder a order.items después de la query original —
# en modo async esto falla con MissingGreenlet, no es solo "más lento"
result = await db.execute(select(Order).where(Order.id == order_id))
order = result.scalar_one()
print(order.items)  # 💥 error

# ✅ eager loading: se trae todo en la query original, no hace falta ningún acceso posterior
result = await db.execute(
    select(Order).options(selectinload(Order.items)).where(Order.id == order_id)
)
order = result.scalar_one()
print(order.items)  # ✅ ya está cargado en memoria
```

## 7. Exception handlers en rutas async

Los exception handlers también pueden (y deben, si hacen I/O) ser `async def`. Si un handler loguea a un servicio externo o a la DB, necesita el mismo cuidado del punto 4: si esa llamada es sync y no lleva `await`, bloquea el event loop igual que bloquearía dentro de una ruta.

```python
@app.exception_handler(OrderNotFoundError)
async def order_not_found_handler(request: Request, exc: OrderNotFoundError):
    await audit_log.record(f"Order {exc.order_id} not found")  # I/O real → necesita await
    return JSONResponse(status_code=404, content={"error": "order_not_found"})
```

## 8. Cómo medir si realmente mejoró algo

No asumir que async = más rápido. La forma real de confirmarlo es comparar throughput y latencia bajo carga concurrente, antes y después de migrar — no en un script local de un solo usuario, donde sync y async se sienten idénticos.

```bash
# ✅ comparar requests/seg y latencia p95 bajo carga real, no adivinar
hey -n 1000 -c 50 http://localhost:8000/orders
```

Si un endpoint no tiene I/O real que esperar (una query trivial a una DB local, por ejemplo), migrarlo a async solo agrega la complejidad de mantener un driver/engine async, sin ninguna ganancia medible.

---
Relacionado: [Endpoints para microservicios](endpoints-microservicios.md), [Diagnóstico Backend](../diagnostico/backend.md) (CPU bound, paralelismo y concurrencia).
