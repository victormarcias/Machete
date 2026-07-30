# Eventos DOM

Cómo se propagan los eventos en el browser — concepto de la plataforma web, no de ningún framework.

## Bubbling vs Capturing

Un evento (ej. un click) atraviesa el DOM en dos fases: primero **capturing** (baja desde `window` hasta el elemento donde ocurrió el click), después **bubbling** (sube de vuelta desde ese elemento hasta `window`). Por default, `addEventListener` escucha en la fase de bubbling.

```js
parent.addEventListener('click', () => console.log('parent'));
child.addEventListener('click', () => console.log('child'));

// click en child imprime: "child" luego "parent" (bubbling, default)

parent.addEventListener('click', () => console.log('parent capturing'), { capture: true });
// con capture: true, este listener corre ANTES de que el evento llegue al target
```

## Delegación de eventos

En vez de poner un listener en cada elemento hijo, se pone **uno solo en el padre** y se aprovecha el bubbling para saber en cuál hijo ocurrió el click (`event.target`). Menos listeners = menos memoria, y funciona automáticamente con hijos agregados después (no hace falta re-attachear nada).

```js
// ❌ un listener por cada <li>, y hay que agregar uno nuevo cada vez que se crea un <li>
document.querySelectorAll('li').forEach(li => li.addEventListener('click', handleClick));

// ✅ un solo listener en el <ul>, funciona para hijos presentes y futuros
document.querySelector('ul').addEventListener('click', (e) => {
  if (e.target.tagName === 'LI') handleClick(e);
});
```

## `preventDefault` vs `stopPropagation`

Se confunden seguido porque ambos "frenan" algo, pero cosas distintas:

- **`preventDefault()`**: cancela el comportamiento default del browser para ese evento (ej. que un `<a>` navegue, que un form haga submit). No afecta la propagación — el evento sigue burbujeando normalmente.
- **`stopPropagation()`**: corta la propagación del evento a los siguientes elementos en la cadena (no sigue burbujeando/capturando). No afecta el comportamiento default del browser.

```js
form.addEventListener('submit', (e) => {
  e.preventDefault();      // ✅ evita el reload de página del submit nativo
  // e.stopPropagation();  // esto NO es necesario para lo anterior — son cosas independientes
});
```

## Nota: `SyntheticEvent` en React

React no attachea un listener nativo a cada elemento — attachea **un solo listener** en la raíz del árbol (delegación de eventos a escala de toda la app) y envuelve el evento nativo en un `SyntheticEvent` con una API consistente entre browsers. La lógica de arriba (bubbling, delegación, `preventDefault`) sigue aplicando igual, solo que React ya la implementa por vos a nivel framework.

---
Relacionado: [Diagnóstico Frontend](../diagnostico/frontend.md) (manejo incorrecto de eventos, debounce/throttle).
