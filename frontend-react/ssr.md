# SSR (Server-Side Rendering)

Renderizar el HTML en el servidor para cada request, en vez de mandar un HTML vacío y dejar que el JS del cliente arme todo. Concepto genérico de arquitectura frontend — lo implementan Next.js, Nuxt, SvelteKit, Remix, cada uno con su propia sintaxis.

## CSR vs SSR

- **CSR (Client-Side Rendering)**: el servidor manda un HTML casi vacío (`<div id="root"></div>`) más un bundle de JS. El browser tiene que descargar y ejecutar ese JS antes de que aparezca cualquier contenido — pantalla en blanco mientras tanto, y nada que un crawler de buscador pueda leer sin ejecutar JS.
- **SSR**: el servidor ejecuta la app y devuelve el HTML **ya renderizado con el contenido real** para esa request. El usuario ve contenido apenas llega la respuesta, sin esperar a que el JS cargue — mejor First Contentful Paint y SEO out-of-the-box.

## Hidratación

El HTML que llega del servidor es contenido estático — todavía no tiene los event listeners de React attacheados. **Hidratación** es el paso donde el JS del cliente "toma control" de ese HTML ya existente y lo vuelve interactivo, sin volver a crear los nodos DOM desde cero (reutiliza lo que el servidor ya mandó).

```jsx
// Ej. con Next.js — el mismo componente corre en el servidor (primer render) y en el cliente (hidratación)
export default function ProductPage({ product }) {
  return <h1>{product.name}</h1>; // en el servidor: HTML real. En el cliente: se hidrata sobre ese HTML.
}

export async function getServerSideProps() {
  const product = await fetchProduct(); // corre en el servidor, en cada request
  return { props: { product } };
}
```

Si el HTML que devuelve el servidor no coincide exactamente con lo que el cliente renderizaría (ej. usar `Date.now()` o `Math.random()` directo en el render), la hidratación falla con un *hydration mismatch* — un bug clásico de SSR.

## SSG, la variante prima

**Static Site Generation**: el mismo concepto de "renderizar en el servidor" pero en **build time**, no por request — el HTML se genera una vez y se sirve igual para todos (con una CDN por delante, ver [CDN](../devops/cdn.md)). Sirve cuando el contenido no depende del usuario ni cambia entre requests (un blog, landing pages); SSR hace falta cuando sí depende (un dashboard con datos del usuario logueado).

## Trade-off

SSR mueve trabajo del cliente al servidor: mejor experiencia inicial, pero cada request ahora cuesta CPU del servidor (en vez de ser un archivo estático servido por una CDN) y agrega la complejidad de la hidratación. No es gratis — es elegir dónde pagar el costo de renderizar.
