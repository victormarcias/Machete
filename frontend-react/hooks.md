# Hooks

A diferencia del resto de los temas de esta carpeta, esto no tiene versión "genérica" — los hooks son una API específica de React (y de los frameworks que copiaron el modelo, como Preact). `useState`/`useEffect`/`memo`/`useMemo`/`useCallback` ya se cubrieron en detalle, con el foco puesto en performance, en [Diagnóstico Frontend](../diagnostico/frontend.md). Acá el foco es el mecanismo por detrás: las reglas y cómo armar hooks propios.

## Reglas de hooks

1. **Solo llamar hooks en el nivel superior** de un componente — nunca dentro de un `if`, un loop, o una función anidada.
2. **Solo llamar hooks desde componentes de React o desde otros hooks** — nunca desde una función JS común.

```jsx
// ❌ rompe la regla 1: si la condición cambia entre renders, el orden de los hooks cambia
function Component({ show }) {
  if (show) {
    const [value, setValue] = useState(0); // a veces se llama, a veces no
  }
}

// ✅ el hook siempre se llama, la condición va adentro
function Component({ show }) {
  const [value, setValue] = useState(0);
  if (show) { /* usar value acá */ }
}
```

**Por qué existe esta regla**: React no identifica cada hook por nombre, sino por el **orden** en que se llaman durante el render — internamente los guarda en una lista y los asocia por posición. Si un hook a veces se llama y a veces no, el orden se desalinea entre renders y React le asigna a un hook el estado que le correspondía a otro.

## Custom hooks

Una función que empieza con `use` y llama a otros hooks adentro — la forma de extraer lógica con estado reutilizable entre componentes, sin duplicar código ni recurrir a patrones más pesados (HOCs, render props).

```jsx
// ✅ custom hook: encapsula fetch + loading + error, reutilizable en cualquier componente
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch(url).then(r => r.json()).then(setData).finally(() => setLoading(false));
  }, [url]);

  return { data, loading };
}

// uso: cualquier componente pide solo lo que necesita, sin repetir la lógica de fetch
function UserProfile({ userId }) {
  const { data, loading } = useFetch(`/api/users/${userId}`);
  if (loading) return <Spinner />;
  return <div>{data.name}</div>;
}
```

Un custom hook no comparte estado entre los componentes que lo usan — cada llamada tiene su propia instancia de `useState`/`useEffect`, como si el código estuviera copiado y pegado (pero sin estarlo).

---
Relacionado: [Diagnóstico Frontend](../diagnostico/frontend.md) (`useEffect`, `memo`/`useMemo`/`useCallback`).
