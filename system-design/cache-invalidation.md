# Cache Invalidation

"Solo hay dos cosas difíciles en Computer Science: cache invalidation y darle nombre a las cosas" — la broma es vieja porque es verdad: cachear es fácil, invalidar en el momento correcto sin servir datos viejos ni tirar la cache entera de más, es lo difícil.

## TTL vs invalidación explícita

- **TTL (Time To Live)**: la entrada expira sola después de X segundos. Simple, no requiere que nadie se acuerde de invalidar nada — pero durante la ventana del TTL, un cambio real puede no reflejarse todavía (dato *stale* por diseño).
- **Invalidación explícita**: el código que escribe el dato también borra/actualiza la entrada de cache correspondiente en el mismo momento. Más preciso (cero ventana de dato viejo), pero requiere acordarse de invalidar en **todos** los lugares que puedan modificar ese dato — un solo lugar que se olvide deja cache desactualizada indefinidamente.

```python
def update_product(id, data):
    db.products.update(id, data)
    cache.delete(f"product:{id}")  # ✅ invalidación explícita en el mismo write
```

## Cache-aside, write-through, write-behind

El patrón **cache-aside** (leer del cache, si no está ir a la DB y cachear el resultado) ya está cubierto con ejemplo de código en [Diagnóstico Backend](../diagnostico/backend.md#falta-de-cache). Las otras dos estrategias escriben distinto:

- **Write-through**: cada escritura va primero al cache, y el cache se encarga de propagarla a la DB de forma sincrónica. El cache nunca queda desactualizado, pero cada write paga la latencia de ambos.
- **Write-behind (write-back)**: la escritura va al cache y se confirma inmediatamente; la propagación a la DB pasa en background, asincrónica. Writes más rápidos, pero hay una ventana donde el dato solo existe en cache — si el cache se cae antes de propagar, se pierde.

## Cache stampede (thundering herd)

Una key muy pedida expira (TTL) y, en el mismo instante, cientos de requests concurrentes la piden — todas ven un cache miss a la vez y le pegan a la DB simultáneamente, como si el cache no existiera. El pico puede tirar abajo la DB justo en el peor momento (la key era popular, por eso estaba cacheada).

**Mitigaciones**:
- *Locking*: el primer request que detecta el miss recalcula y cachea; los demás esperan ese resultado en vez de ir todos a la DB.
- *Early recomputation*: recalcular la key un poco antes de que expire, para que nunca haya una ventana sin cache.

```python
# ✅ lock simple: solo un request recalcula, el resto espera
def get_product(id):
    cached = cache.get(f"product:{id}")
    if cached:
        return cached
    with lock(f"lock:product:{id}"):          # los demás requests esperan acá
        cached = cache.get(f"product:{id}")    # re-chequear: quizás ya lo cacheó otro mientras esperaba
        if cached:
            return cached
        result = db.products.find(id)
        cache.set(f"product:{id}", result, ttl=300)
        return result
```

---
Relacionado: [Diagnóstico Backend](../diagnostico/backend.md#falta-de-cache) (cache-aside), [Consistencia](atributos-de-calidad.md#consistencia).
