# Performance Diagnostics

Herramientas para medir en vez de adivinar dónde está el problema de performance de una app frontend.

## Lighthouse (genérico)

Auditoría automatizada que corre en cualquier página (Chrome DevTools, CLI, o CI) y devuelve un score de Performance, Accessibility, Best Practices y SEO, con recomendaciones puntuales. Mide los **Core Web Vitals**:

- **LCP** (Largest Contentful Paint) — cuánto tarda en pintarse el elemento más grande visible (la métrica de "se siente cargado").
- **INP** (Interaction to Next Paint) — cuánto tarda la UI en responder después de una interacción del usuario.
- **CLS** (Cumulative Layout Shift) — cuánto "salta" el layout mientras carga (imágenes sin `width`/`height` reservado, por ejemplo).

```bash
npx lighthouse https://miapp.com --view
```

## Bundle analyzer (genérico)

Visualiza qué hay realmente adentro del bundle final de JS — un treemap donde cada caja es un módulo, su tamaño proporcional al espacio que ocupa. Sirve para encontrar la dependencia de 200kb que se importó por una sola función (ver el ejemplo de `lodash` completo vs `lodash/debounce` en [Diagnóstico Frontend](../diagnostico/frontend.md)).

```bash
# Ej. con Webpack
npx webpack-bundle-analyzer stats.json

# Ej. con Vite/Rollup
npx vite-bundle-visualizer
```

## React Profiler (específico de React)

Tab del React DevTools que graba una sesión de renders y muestra, por componente, cuánto tardó cada uno y **por qué** se re-renderizó (qué prop o qué state cambió). Es la herramienta puntual para responder "¿por qué este componente se está re-renderizando tanto?" en vez de adivinar mirando el código.

Los otros dos (Lighthouse, bundle analyzer) dicen **qué tan grande/lenta es la carga inicial**; el Profiler dice **qué está pasando durante la interacción**, ya con la app corriendo.

---
Relacionado: [Diagnóstico Frontend](../diagnostico/frontend.md), [Tree shaking](tree-shaking.md).
