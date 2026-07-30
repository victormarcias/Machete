# Module Federation

Arquitectura de **micro-frontends**: dividir una app frontend grande en piezas independientes, cada una con su propio build y deploy, que se componen recién en **runtime** en vez de todas juntas en un único bundle.

## El problema que resuelve

Una SPA tradicional es un monolito de frontend: todo el código vive en un solo repo, un solo build, un solo deploy — si diez equipos trabajan sobre la misma app, todos comparten el mismo pipeline y cualquier release espera a que todo esté listo. Micro-frontends le aplican a la UI la misma idea que microservicios le aplica al backend: cada equipo dueño de una parte de la pantalla, con su propio ciclo de desarrollo y deploy.

## Cómo funciona (Ej. con Webpack Module Federation)

Una app (`host`) puede cargar en runtime un módulo servido por otra app (`remote`), como si fuera un import normal — pero ese código ni siquiera existía en el bundle del host al momento de buildearlo.

```js
// webpack.config.js del remote (expone un componente)
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'checkout',
      filename: 'remoteEntry.js',
      exposes: { './CheckoutButton': './src/CheckoutButton' },
    }),
  ],
};

// webpack.config.js del host (consume el remote)
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      remotes: { checkout: 'checkout@https://checkout.miapp.com/remoteEntry.js' },
    }),
  ],
};
```

```jsx
// en el código del host, se importa como si fuera local —
// pero en runtime lo trae desde el servidor del equipo de checkout
const CheckoutButton = React.lazy(() => import('checkout/CheckoutButton'));
```

## Trade-offs

No es gratis: agrega **acoplamiento en runtime** (si el remote está caído o cambió su contrato, el host se rompe en producción, no en build time), *version skew* entre equipos (el host puede estar corriendo contra una versión del remote distinta a la que testeó), y complejidad extra de testing (probar el sistema completo requiere levantar varios remotes a la vez). Para equipos chicos, o cuando el problema es solo "compartir componentes" y no "deploys independientes", una librería compartida en un monorepo suele alcanzar sin pagar ese costo.
