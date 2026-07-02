# Diagnóstico Frontend

Causas más comunes de lentitud en el cliente (React), de más a menos frecuentes.

## `useEffect` mal usado → re-renders en cadena

Dependencias mal declaradas (o ausentes) provocan que el efecto se dispare de más, y cada disparo puede setear estado y forzar otro render.

```jsx
// ❌ efecto sin array de dependencias: corre en cada render
useEffect(() => { fetchData(); });

// ✅ solo corre cuando cambia userId
useEffect(() => { fetchData(); }, [userId]);
```

Revisar con el **React DevTools Profiler**: cuántas veces se renderiza un componente y por qué (highlight de "why did this render").

## Falta de `key` en listas

Sin `key` estable (o usando el índice del array como key cuando la lista se reordena/filtra), React no puede diffear eficientemente y re-renderiza/remonta más de lo necesario.

```jsx
// ❌ key = index, se rompe si la lista se reordena
items.map((item, i) => <Item key={i} {...item} />)

// ✅ key = id estable del dato
items.map((item) => <Item key={item.id} {...item} />)
```

## Falta de paginación / virtualización

Renderizar miles de nodos DOM de una sola vez satura el layout/paint. Paginar, o usar *windowing* (`react-window`, `react-virtualized`) para listas largas — solo se montan los items visibles en viewport.

```jsx
import { FixedSizeList } from 'react-window';

// ✅ solo monta las filas visibles, no las 10.000 del array
<FixedSizeList height={600} itemCount={items.length} itemSize={40}>
  {({ index, style }) => <div style={style}>{items[index].name}</div>}
</FixedSizeList>
```

## Componentes que no usan `memo`/`useMemo`/`useCallback`

Sin memoización, un componente hijo se re-renderiza aunque sus props no hayan cambiado (si el padre re-renderiza), o un cálculo pesado se repite en cada render.

```jsx
const total = useMemo(() => calcularTotalPesado(items), [items]);
const handleClick = useCallback(() => doSomething(id), [id]);
const Row = memo(function Row({ item }) { ... });
```

No abusar: memoizar todo agrega overhead de comparación — usarlo donde el costo evitado (render caro, prop que rompe referencia) lo justifica.

## Manejo incorrecto de eventos

Handlers creados inline en cada render (`onClick={() => f(id)}`) generan una función nueva cada vez, invalidando la memoización de hijos que dependen de esa prop. Falta de *debounce/throttle* en eventos de alta frecuencia (`scroll`, `resize`, `input` de búsqueda) dispara trabajo excesivo.

```ts
// ✅ debounce con TypeScript, evita 1 request por cada tecla
function debounce<T extends (...args: any[]) => void>(fn: T, ms: number) {
  let timer: ReturnType<typeof setTimeout>;
  return (...args: Parameters<T>) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), ms);
  };
}
const onSearch = debounce((q: string) => fetchResults(q), 300);
```

## Cálculos complejos en el render / re-renders excesivos

Cualquier cómputo pesado (`sort`, `filter`, transformaciones) ejecutado directo en el cuerpo del componente se repite en cada render. Sacarlo con `useMemo` o moverlo fuera del componente si no depende de props/state.

```jsx
// ❌ se ordena en cada render, aunque items no haya cambiado
const sorted = items.sort((a, b) => a.price - b.price);

// ✅ solo se recalcula cuando items cambia
const sorted = useMemo(() => [...items].sort((a, b) => a.price - b.price), [items]);
```

## Texturas / imágenes pesadas

Imágenes sin optimizar (tamaño real, formato) bloquean paint y consumen ancho de banda. Usar formatos modernos (WebP/AVIF), `srcset`/`loading="lazy"`, y CDN de imágenes con resize on-the-fly.

```jsx
// ✅ carga diferida + tamaño responsive
<img src="foto-800.webp" srcSet="foto-400.webp 400w, foto-800.webp 800w" loading="lazy" alt="" />
```

## Bloqueo del hilo principal

JS es single-threaded: un cálculo largo (parseo, loops pesados) congela la UI. Mover trabajo pesado a un **Web Worker**, o trocearlo con `requestIdleCallback`/chunking.

```js
// ✅ el cálculo pesado corre fuera del hilo principal
const worker = new Worker('heavy-calc.worker.js');
worker.postMessage(bigDataset);
worker.onmessage = (e) => setResult(e.data);
```

## Muchas actualizaciones de estado

Múltiples `setState` seguidos sin batching (fuera de handlers de React, ej. en `setTimeout`/`fetch` callbacks en versiones viejas de React) disparan un render por cada uno. React 18+ hace *automatic batching* en la mayoría de los casos, pero conviene revisar.

```jsx
// ❌ (pre-React 18, fuera de un handler) dispara 2 renders
setTimeout(() => {
  setCount(c => c + 1);
  setFlag(true);
}, 1000);

// ✅ agrupar el estado relacionado en un solo objeto/reducer evita el problema de raíz
const [state, setState] = useState({ count: 0, flag: false });
setTimeout(() => setState(s => ({ count: s.count + 1, flag: true })), 1000);
```

## Uso de componentes no optimizados (de terceros)

Librerías de UI pesadas importadas completas en vez de tree-shakeable, o componentes que no soportan `memo` internamente. Ver [tree shaking](../frontend-react/README.md) y auditar bundle con bundle-analyzer.

```ts
// ❌ importa toda la librería (~70kb) para usar una función
import _ from 'lodash';
_.debounce(fn, 300);

// ✅ importa solo lo necesario, tree-shakeable
import debounce from 'lodash/debounce';
debounce(fn, 300);
```

---
Para medir en vez de adivinar: React DevTools **Profiler**, **Lighthouse**, y **bundle-analyzer** — ver [frontend-react](../frontend-react/README.md).
