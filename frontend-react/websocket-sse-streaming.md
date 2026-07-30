# WebSocket / SSE / Streaming

Tres formas de mandar datos del servidor al cliente más allá del request/response clásico de HTTP, cada una resolviendo un problema distinto.

## WebSocket — bidireccional

Conexión persistente y full-duplex: una vez abierta, tanto cliente como servidor pueden mandar mensajes en cualquier momento, sin que uno tenga que "pedirle" al otro. Empieza como un request HTTP normal que hace *upgrade* de protocolo.

```js
const socket = new WebSocket('wss://api.miapp.com/chat');
socket.onmessage = (event) => console.log('Recibido:', event.data);
socket.send('hola');  // el cliente también puede iniciar el envío en cualquier momento
```

**Cuándo usarlo**: chat, edición colaborativa en tiempo real, cualquier caso donde el cliente también necesita empujar datos por el mismo canal (no solo recibir).

## SSE (Server-Sent Events) — unidireccional

El servidor manda eventos al cliente sobre una conexión HTTP normal (sin upgrade de protocolo) — el cliente **no puede** mandar datos por ese mismo canal, solo recibir. Más simple que WebSocket cuando no hace falta el sentido inverso, y el browser reconecta automáticamente si se corta.

```js
const events = new EventSource('/api/notifications');
events.onmessage = (event) => console.log('Notificación:', event.data);
// no hay events.send() — SSE es estrictamente servidor → cliente
```

**Cuándo usarlo**: notificaciones live, feeds, progreso de una tarea larga — cualquier caso donde el flujo de datos es solo servidor → cliente.

## Streaming (fetch + `ReadableStream`)

Una respuesta HTTP normal, pero enviada en pedazos (*chunked*) a medida que el servidor los va generando, en vez de esperar a tener la respuesta completa. No es un protocolo aparte como WebSocket/SSE — es la misma request de siempre, leída incrementalmente.

```js
const response = await fetch('/api/generate');
const reader = response.body.getReader();

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  console.log(new TextDecoder().decode(value)); // cada chunk apenas llega, no al final
}
```

**Cuándo usarlo**: el caso típico hoy es texto generado por un LLM token por token — el usuario ve la respuesta aparecer progresivamente en vez de esperar a que termine todo el output.

## Comparación

| | WebSocket | SSE | Streaming |
|---|---|---|---|
| Dirección | Bidireccional | Servidor → cliente | Servidor → cliente |
| Protocolo | Upgrade a `ws://` | HTTP normal | HTTP normal |
| Reconexión automática | No (hay que implementarla) | Sí, nativa del browser | No aplica (una request, un stream) |
| Caso típico | Chat, colaboración en vivo | Notificaciones, feeds | Respuestas largas generadas incrementalmente |
