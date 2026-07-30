# Patrón Observer en el browser

El patrón Observer en general (un objeto notifica a una lista de dependientes cuando cambia) ya está cubierto en [Patrones de comportamiento](../system-design/patrones-comportamiento.md#observer). Acá el ángulo específico de frontend: el browser trae el patrón ya resuelto como APIs nativas, reemplazando código que antes se resolvía con polling manual (y su costo de performance).

## `MutationObserver`

Notifica cuando cambia el DOM (se agregan/quitan nodos, cambia un atributo), sin tener que revisar el DOM a mano en un intervalo.

```js
const observer = new MutationObserver((mutations) => {
  mutations.forEach(m => console.log('Cambió:', m.type));
});

observer.observe(document.getElementById('lista'), { childList: true, attributes: true });
// ✅ se entera de cada cambio apenas ocurre — no hay setInterval comparando el DOM cada X ms
```

## `IntersectionObserver`

Notifica cuando un elemento entra o sale del viewport (o de otro elemento contenedor). Reemplaza el patrón viejo de escuchar el evento `scroll` y calcular manualmente posiciones — mucho más caro en performance porque `scroll` dispara decenas de veces por segundo.

```js
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) cargarImagen(entry.target); // lazy loading real
  });
});

document.querySelectorAll('img[data-src]').forEach(img => observer.observe(img));
```

**Caso de uso típico**: lazy loading de imágenes, infinite scroll, animaciones que arrancan al entrar en pantalla — todo sin un solo listener de `scroll`. Ver también [Debounce / Throttle](../diagnostico/frontend.md#manejo-incorrecto-de-eventos), la alternativa cuando sí hace falta escuchar `scroll` directamente.

## `ResizeObserver`

Notifica cuando cambia el tamaño de un elemento — no de la ventana entera (`window.resize`), sino de un elemento puntual, incluso si cambió por CSS (un flex/grid que se reacomodó) sin que la ventana se haya tocado.

```js
const observer = new ResizeObserver((entries) => {
  entries.forEach(entry => console.log('Nuevo tamaño:', entry.contentRect.width));
});

observer.observe(document.querySelector('.panel-redimensionable'));
```

## El patrón común

Los tres reemplazan la misma alternativa mala: revisar algo en un loop (`setInterval`) o reaccionar a un evento demasiado ruidoso (`scroll`, `resize` de window) para detectar un cambio puntual. La API nativa notifica solo cuando el cambio específico ocurre — es Observer aplicado a la plataforma web en vez de a una clase propia.
