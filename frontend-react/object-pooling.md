# Object Pooling

Reutilizar un conjunto fijo de objetos en vez de crearlos y destruirlos constantemente. Concepto genérico (viene de game dev), pero aplica directo a cualquier loop de alta frecuencia en frontend: animaciones, canvas, sistemas de partículas.

## Por qué importa

Crear un objeto nuevo en cada frame de una animación (60 veces por segundo) genera basura que el garbage collector tiene que limpiar — y esa limpieza puede pausar el hilo principal justo en medio de una animación, generando el *jank* (tirones visibles) que se nota como frames perdidos.

```js
// ❌ una partícula nueva por frame, miles de objetos descartados por segundo
function spawnParticle() {
  return { x: 0, y: 0, vx: Math.random(), vy: Math.random() };
}
function tick() {
  particles.push(spawnParticle()); // el GC tiene que limpiar las viejas descartadas
}
```

## El patrón

Se pre-crea un conjunto fijo de objetos al arrancar, y en vez de `new`/descartar, se "pide prestado" uno inactivo del pool y se "devuelve" cuando ya no hace falta.

```js
class ParticlePool {
  constructor(size) {
    this.pool = Array.from({ length: size }, () => ({ x: 0, y: 0, vx: 0, vy: 0, active: false }));
  }

  acquire() {
    const p = this.pool.find(p => !p.active);
    if (p) p.active = true;
    return p; // reutiliza un objeto existente, no crea uno nuevo
  }

  release(p) {
    p.active = false; // vuelve al pool, listo para la próxima partícula
  }
}

const pool = new ParticlePool(500);
const particle = pool.acquire();
// ... usar la partícula ...
pool.release(particle); // no se descarta, se recicla
```

Cero allocations nuevas en el loop caliente — el pool ya tiene todo lo que va a necesitar reservado de entrada.

## El mismo principio en las listas virtualizadas

[Virtualización / Windowing](../diagnostico/frontend.md#falta-de-paginación--virtualización) aplica la misma idea a nivel de nodos DOM: en vez de montar y desmontar un `<div>` por cada fila que entra/sale del viewport al scrollear, las librerías de virtualización reciclan un puñado fijo de nodos DOM y solo les cambian el contenido — el DOM también es "caro" de crear/destruir, igual que los objetos en memoria de un game loop.
