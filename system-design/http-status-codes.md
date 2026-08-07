# HTTP Status Codes

El primer dígito define la categoría; ese dígito solo ya le dice al cliente cómo interpretar la respuesta sin necesidad de leer el body.

## 1xx — Informational

Respuestas provisorias, antes de la respuesta final — el cliente casi nunca las maneja directo, las resuelve la capa HTTP por debajo.

- **100 Continue**: el servidor confirma que puede recibir el resto de un request grande antes de que el cliente lo mande completo (evita subir un body enorme para que recién ahí el servidor lo rechace).
- **101 Switching Protocols**: confirma el upgrade de protocolo — es el mecanismo por el que arranca una conexión WebSocket (ver [WebSocket / SSE / Streaming](../frontend-react/websocket-sse-streaming.md)).

## 2xx — Success

- **200 OK**: éxito genérico, con body en la respuesta. El default para casi cualquier `GET`/`PUT`/`PATCH` exitoso.
- **201 Created**: el request creó un recurso nuevo (`POST` típicamente). La respuesta debería incluir dónde vive ese recurso (header `Location`, o el id en el body).
- **202 Accepted**: el request se aceptó pero el procesamiento es asíncrono — todavía no terminó. Típico en colas de trabajo (ver [Idempotencia](atributos-de-calidad.md#idempotencia) para el caso de reintentos sobre este tipo de request).
- **204 No Content**: éxito, pero no hay nada que devolver en el body — común en `DELETE` o en un `PUT` donde el cliente ya tiene el dato actualizado.

```
POST /orders          → 201 Created   (Location: /orders/42)
DELETE /orders/42      → 204 No Content
GET /orders/42          → 200 OK
```

## 3xx — Redirection

- **301 Moved Permanently**: el recurso se mudó para siempre — los browsers y crawlers cachean este redirect agresivamente, actualizan sus links.
- **302 Found**: redirect temporal — el recurso sigue ahí, esta vez apunta para otro lado, pero no hay que actualizar nada permanentemente.
- **304 Not Modified**: respuesta a un request condicional (`If-None-Match`/`If-Modified-Since`) — le dice al cliente "tu copia cacheada sigue siendo válida, no te mando el body de nuevo". La base de la cache HTTP (ver [CDN](../devops/cdn.md)).

## 4xx — Client Error

El error es del lado del cliente — el request está mal formado, no autenticado, no autorizado, o pide algo que no existe.

- **400 Bad Request**: el request está malformado o no pasa la validación (falta un campo requerido, un tipo inválido).
- **401 Unauthorized**: no está autenticado — falta el token, o es inválido/expiró.
- **403 Forbidden**: está autenticado, pero no tiene permiso para esta acción. Ver la distinción completa en [Autenticación vs Autorización](autenticacion.md#7-autenticación-vs-autorización) — es el error más confundido de toda la lista.
- **404 Not Found**: el recurso no existe. También se usa a veces **a propósito** en vez de `403`, para no filtrarle a un atacante que un recurso existe pero no tiene permiso — depende de cuánta información querés exponer.
- **405 Method Not Allowed**: el recurso existe, pero no soporta ese verbo HTTP (ej. `DELETE /orders` cuando esa ruta solo acepta `GET`/`POST`).
- **409 Conflict**: el request es válido, pero choca con el estado actual del recurso — el caso típico es un update basado en una versión vieja (ver [Optimistic locking](../sql/locks.md#pessimistic-vs-optimistic-locking)).
- **422 Unprocessable Entity**: el request tiene el formato correcto (JSON válido) pero los datos no pasan las reglas de negocio/validación — la línea con `400` es fina y varía según el equipo; muchos frameworks (FastAPI incluido) usan `422` específicamente para errores de validación de schema.
- **429 Too Many Requests**: rate limit excedido — normalmente viene con un header `Retry-After` indicando cuánto esperar antes de reintentar.

## 5xx — Server Error

El error es del lado del servidor — el cliente hizo todo bien, algo se rompió del otro lado.

- **500 Internal Server Error**: error genérico no manejado — una excepción que se escapó sin un handler específico (ver [Manejo de excepciones](../python/endpoints-microservicios.md#5-manejo-de-excepciones-con-appexception_handler)).
- **502 Bad Gateway**: un proxy/load balancer recibió una respuesta inválida del servidor de origen (el servidor de atrás está caído o devolvió basura).
- **503 Service Unavailable**: el servidor está temporalmente no disponible (sobrecargado, en mantenimiento) — normalmente con `Retry-After`.
- **504 Gateway Timeout**: un proxy/load balancer esperó demasiado la respuesta del servidor de origen y se rindió.

**502 vs 503 vs 504**: los tres los suele devolver el load balancer, no la app — `502` es "el backend contestó algo que no entiendo o no contestó nada válido", `503` es "el backend no está aceptando conexiones ahora mismo", `504` es "el backend nunca contestó a tiempo". Distinguirlos ayuda a saber dónde mirar: `504` apunta a lentitud (ver [Diagnóstico Backend](../diagnostico/backend.md)), `502`/`503` apuntan a que el proceso está caído o no arrancó.

---
Relacionado: [Autenticación vs Autorización](autenticacion.md#7-autenticación-vs-autorización), [Idempotencia](atributos-de-calidad.md#idempotencia), [Endpoints para microservicios](../python/endpoints-microservicios.md).
