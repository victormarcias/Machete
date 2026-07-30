# CDN (Content Delivery Network)

Red de servidores (*edge nodes*) distribuidos geográficamente que cachean copias de assets estáticos cerca de cada usuario, para no tener que viajar hasta el servidor de origin en cada request.

Sin CDN, un usuario en Argentina pidiendo una imagen de un servidor en EE.UU. paga esa latencia geográfica en **cada** request, aunque el archivo nunca cambie. Con CDN, la primera visita a una región cachea el archivo en el edge node más cercano; el resto de los usuarios de esa región lo reciben desde ahí.

```
Cache-Control: public, max-age=31536000, immutable
```

Ese header le dice a la CDN (y al browser) que puede cachear el asset por un año — seguro cuando el archivo tiene un hash en el nombre (`app.a1b2c3.js`) y por lo tanto un cambio en el contenido siempre genera una URL nueva, no una actualización silenciosa de la vieja.

**Invalidación**: si hay que forzar que la CDN descarte una copia vieja antes de que expire el TTL (ej. un deploy urgente), se hace con un *purge*/*invalidation* explícito contra la API del proveedor — es la excepción, no el flujo normal.

---
Relacionado: [Diagnóstico DevOps](../diagnostico/devops.md) (síntoma: red lenta por falta de CDN).
