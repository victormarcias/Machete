# Tree Shaking

El bundler elimina del bundle final el código exportado que nadie importa. Concepto genérico de cualquier bundler moderno (Webpack, Rollup, esbuild, Vite) — no es específico de React.

## Por qué funciona con ES Modules y no con CommonJS

Los `import`/`export` de ES Modules son **estáticos**: el bundler puede analizar el código sin ejecutarlo y saber exactamente qué exports de cada módulo se usan en algún lado. `require()` de CommonJS es **dinámico** (puede llamarse condicionalmente, con un string armado en runtime) — el bundler no puede garantizar de antemano qué se va a necesitar, así que no puede descartar nada con seguridad.

```js
// ✅ ES Modules: el bundler ve estáticamente que solo se usa `formatDate`
import { formatDate } from './utils';

// ❌ CommonJS: en teoría el path podría variar en runtime, el bundler no puede analizarlo igual
const utils = require('./utils');
```

Ver el ejemplo completo con `lodash` (import de la librería completa vs. `lodash/debounce`) en [Diagnóstico Frontend](../diagnostico/frontend.md#uso-de-componentes-no-optimizados-de-terceros).

## `sideEffects` en `package.json`

El bundler no puede eliminar un módulo si ejecutarlo tiene efectos secundarios (ej. modifica un objeto global, registra algo) aunque nada importe sus exports — por las dudas, lo deja. Una librería puede declarar explícitamente que sus archivos **no** tienen side effects, para que el bundler los pueda descartar con confianza si no se usan.

```json
{
  "name": "mi-libreria",
  "sideEffects": false
}
```

```json
// o una lista puntual de los archivos que sí los tienen (ej. un polyfill que se auto-ejecuta)
{
  "sideEffects": ["./src/polyfills.js"]
}
```

Sin esta declaración, el bundler asume que **todo** puede tener side effects y es más conservador — descarta menos código del que en realidad podría.

---
Relacionado: [Diagnóstico Frontend](../diagnostico/frontend.md), [Performance Diagnostics](performance-diagnostics.md) (bundle analyzer, para ver qué quedó adentro).
