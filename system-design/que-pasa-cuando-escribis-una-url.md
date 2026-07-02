# Qué pasa cuando escribís una URL

Pregunta clásica de entrevista de system design: recorrido completo desde que apretás Enter hasta que la página está renderizada en pantalla. Sirve para mostrar que entendés todas las capas — red, servidor, y cliente — no solo una.

## 1. Parseo de la URL

El navegador separa la URL en sus partes: protocolo (`https`), host (`www.ejemplo.com`), puerto (default 443 para https, 80 para http), path (`/productos`), query string (`?id=1`).

```
https://www.ejemplo.com:443/productos?id=1
└─┬──┘   └───────┬───────┘└┬┘└───┬───┘└──┬──┘
protocolo       host      puerto path   query
```

## 2. Resolución DNS — de dominio a IP

El host (`www.ejemplo.com`) no sirve para conectarse por red, hace falta una IP. El navegador busca en cascada, deteniéndose en el primer lugar donde encuentra la respuesta:

1. **Cache del navegador** — ¿ya resolví este dominio hace poco?
2. **Cache del sistema operativo**.
3. **Router local**.
4. **Resolver DNS del ISP** (o uno público como `8.8.8.8`, `1.1.1.1`) — si nadie de los anteriores tiene la respuesta, este hace la resolución recursiva completa:
   - Pregunta a un **root server** (`.`) → le dice dónde están los servidores del TLD (`.com`).
   - Pregunta al **TLD server** (`.com`) → le dice dónde está el **authoritative nameserver** de `ejemplo.com`.
   - Pregunta al **authoritative nameserver** → devuelve la IP real (registro `A`/`AAAA`).
5. La IP se cachea en cada nivel según el **TTL** del registro DNS, para no repetir todo este viaje en la próxima visita.

Este paso puede agregar decenas a cientos de ms si nada está cacheado — es una de las primeras cosas a mirar cuando una app "arranca lenta". Ver [Cold start](../diagnostico/devops.md) y [CDN](../diagnostico/devops.md) para latencia relacionada.

## 3. Conexión TCP — three-way handshake

Con la IP en mano, el navegador abre una conexión TCP con el servidor (puerto 443/80):

```
Cliente → SYN      → Servidor
Cliente ← SYN-ACK  ← Servidor
Cliente → ACK      → Servidor
```

Tres viajes de red (o 1.5 *round trips*) antes de poder mandar un solo byte de datos. Con **HTTP/2** o **HTTP/3** (QUIC, sobre UDP) se reduce este costo reutilizando conexiones o evitando el handshake TCP tradicional.

## 4. TLS handshake (si es HTTPS)

Encima del TCP ya abierto, se negocia la capa de cifrado:

1. Cliente manda `ClientHello` (versiones de TLS y cifrados que soporta).
2. Servidor responde con su **certificado** (incluye la clave pública) y el cifrado elegido.
3. Cliente valida el certificado contra una **autoridad certificadora (CA)** de confianza, verifica que el dominio coincida y que no esté expirado/revocado.
4. Se negocia una clave de sesión simétrica (más rápida que la asimétrica) que se usa para cifrar el resto de la comunicación.

TLS 1.3 redujo esto a 1 round trip (contra 2 de TLS 1.2). Este paso es el que garantiza que nadie en el medio (Wi-Fi público, ISP) pueda leer o modificar los datos.

## 5. Request HTTP

Con la conexión cifrada lista, el navegador manda la request:

```http
GET /productos?id=1 HTTP/1.1
Host: www.ejemplo.com
User-Agent: Mozilla/5.0 ...
Accept: text/html
Cookie: session=abc123
```

## 6. El servidor procesa la request

Del lado del servidor puede haber varias capas antes de llegar a una respuesta:

- **Load balancer** (L4/L7) recibe la conexión y decide a qué servidor de aplicación mandarla.
- **Servidor de aplicación** ejecuta la lógica: autenticación, queries a la base de datos, llamadas a otros servicios.
- Posiblemente pega contra **cache** (Redis) antes de ir a la base de datos.
- Arma la respuesta (HTML renderizado en servidor, JSON, etc.).

Ver [Load balancers](README.md) y [Diagnóstico Backend](../diagnostico/backend.md)/[Diagnóstico Base de Datos](../diagnostico/base-de-datos.md) para qué puede salir mal en este paso.

## 7. Response HTTP

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Cache-Control: public, max-age=3600
Content-Length: 5324

<!DOCTYPE html>...
```

El `status code` le dice al navegador cómo interpretar la respuesta (`200` OK, `301`/`302` redirect, `404` not found, `500` error de servidor).

## 8. El navegador renderiza

Con el HTML en mano, arranca el **critical rendering path**:

1. **Parseo de HTML → DOM tree**: el navegador arma el árbol de nodos del documento. Al encontrar un `<link rel="stylesheet">` o un `<script>`, dispara requests adicionales (reutilizando la conexión si es posible — *keep-alive*, o multiplexadas si es HTTP/2).
2. **Parseo de CSS → CSSOM**: árbol de estilos calculados.
3. **DOM + CSSOM → Render Tree**: combina ambos, descartando nodos que no se muestran (`display: none`).
4. **Layout (reflow)**: calcula la posición y tamaño exacto de cada elemento en la pantalla.
5. **Paint**: convierte el render tree en píxeles reales.
6. **Composite**: combina las distintas capas (ej. elementos con `position: fixed`, animaciones en GPU) en la imagen final.

Un `<script>` sin `defer`/`async` **bloquea** el parseo del HTML hasta descargarse y ejecutarse — por eso la recomendación clásica de poner scripts al final del `<body>`, o usar `defer` (descarga en paralelo, ejecuta después del parseo del HTML) / `async` (descarga en paralelo, ejecuta apenas está listo, sin orden garantizado).

```html
<script src="app.js" defer></script>
```

## Resumen del recorrido

```
URL → parseo → DNS (cache → resolver → root → TLD → authoritative)
    → TCP handshake → TLS handshake → HTTP request
    → [load balancer → app server → cache/DB] → HTTP response
    → HTML parse (DOM) + CSS parse (CSSOM) → Render Tree → Layout → Paint → Composite
```

Cada tramo es un punto de diagnóstico distinto — ver [diagnóstico por capa](../diagnostico/README.md) para profundizar en cada uno cuando algo anda lento.
