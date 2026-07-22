# Autenticación y Seguridad — Conceptos Generales

Fundamentos que hay que tener claros antes de escribir una sola línea de login — aplican en cualquier lenguaje o framework. La implementación concreta en Python/FastAPI está en [Autenticación en FastAPI](../python/autenticacion-fastapi.md).

## 1. Hashing vs Encriptado vs Encoding

Tres cosas que suenan parecido y se confunden seguido, pero resuelven problemas distintos:

- **Encoding** (Base64, URL encoding): reversible, sin ninguna clave — es solo una transformación de formato, no seguridad. Cualquiera puede decodificarlo.
- **Encriptado** (AES, etc.): reversible **con una clave**. Se usa cuando en algún momento vas a necesitar recuperar el dato original (ej. un número de tarjeta que hay que mostrar parcialmente, o el tráfico TLS de toda la sesión).
- **Hashing** (SHA-256, bcrypt, Argon2): **irreversible**, una función de un solo sentido. Se usa cuando nunca vas a necesitar el dato original — solo verificar que dos valores coinciden. Para contraseñas, es la única opción correcta.

```python
import base64, hashlib

# Encoding: reversible sin ninguna clave — no es seguridad, es solo un formato
encoded = base64.b64encode(b"password123")
base64.b64decode(encoded)  # devuelve "password123" tal cual, sin necesitar ningún secreto

# Hashing: irreversible — no existe una función "unhash()"
hashed = hashlib.sha256(b"password123").hexdigest()
# no hay forma de recuperar "password123" a partir de hashed —
# solo se puede volver a hashear un intento y comparar los hashes
```

## 2. Por qué nunca se "desencripta" una contraseña

Si un sistema puede recuperar la contraseña original a partir de lo que guardó (porque la "encriptó" en vez de hashearla), eso ya es una falla de diseño: si el servidor se compromete, **todas** las contraseñas quedan expuestas en texto plano. Con hashing, ni siquiera el propio servidor puede recuperar el password original después del registro — en el login se vuelve a hashear el intento y se comparan los hashes.

```python
# ❌ diseño incorrecto: si es reversible ("encriptado"), quien tenga la clave recupera el original
password_guardada = encrypt(password, server_secret_key)

# ✅ diseño correcto: hash irreversible — ni el servidor puede recuperar el password original
password_hash = hash(password)  # en login: hash(intento) == password_hash ?
```

## 3. Salt

Sin salt, la misma contraseña siempre produce el mismo hash — lo que habilita **rainbow tables**: tablas precalculadas de hash → contraseña para millones de passwords comunes. Si un atacante consigue la base de hashes, busca cada uno en la tabla en vez de tener que calcular nada. El **salt** es un valor aleatorio único por usuario que se combina con la contraseña antes de hashear, y se guarda junto al hash (no es secreto) — hace que precalcular una tabla universal sea inútil, porque el espacio de hashes de cada usuario es distinto aunque dos tengan la misma contraseña.

```python
# sin salt: dos usuarios con la misma password → mismo hash, visible en la DB
hash("1234") == hash("1234")  # True — un atacante ve que ambos usuarios comparten password

# con salt: el mismo password produce hashes distintos por usuario
hash("1234" + salt_user_a) != hash("1234" + salt_user_b)  # True
```

## 4. Por qué los algoritmos de hash de passwords son lentos a propósito

`SHA-256`/`MD5` están diseñados para ser **rápidos** — miles de millones de hashes por segundo en una GPU. Excelente para checksums de archivos, pésimo para contraseñas: si un atacante roba la base de hashes, puede probar miles de millones de contraseñas por segundo. `bcrypt`, `scrypt` y `Argon2` son deliberadamente **lentos** (con un "work factor" configurable), y algunos (`scrypt`, `Argon2`) además son *memory-hard* — requieren mucha RAM por intento, lo que encarece muchísimo atacar con GPUs/ASICs en paralelo. Esta es la razón concreta por la que `hashlib.sha256(password)` solo, sin más, **nunca** es aceptable para passwords.

## 5. JWT — estructura y "stateless"

Un JWT tiene tres partes separadas por puntos: `header.payload.signature`, cada una codificada en Base64URL — **no encriptadas**. Cualquiera con el token puede leer el payload completo (es un error común de principiante asumir que es secreto) — la firma solo garantiza que **no fue modificado**, no que sea confidencial. Por eso nunca va una contraseña o un secreto en el payload.

"Stateless" significa que el servidor no guarda ninguna sesión: solo verifica la firma con su clave (secreta o pública, según el algoritmo) y confía en el contenido si la firma es válida. Esto es lo que permite escalar horizontalmente sin pegarle a un store compartido en cada request — ver el ejemplo de sesión en Redis vs sesión en memoria en [Escalabilidad](atributos-de-calidad.md).

## 6. Access token vs Refresh token

- **Access token**: vida corta (minutos), se manda en cada request, es lo que valida cada endpoint protegido.
- **Refresh token**: vida larga (días/semanas), se guarda con más cuidado, y se usa solo para pedir un access token nuevo cuando el anterior expira — sin forzar al usuario a loguearse de nuevo.

Separarlos acota la ventana de daño: si un access token se filtra, expira en minutos; el refresh token (más sensible, de vida más larga) viaja con mucha menos frecuencia, reduciendo su exposición.

## 7. Autenticación vs Autorización

- **Autenticación**: ¿quién sos? (verificar identidad — el login).
- **Autorización**: ¿qué podés hacer? (permisos/roles — ocurre después de autenticar).

El nombre de los status codes HTTP confunde esto seguido: `401 Unauthorized` en realidad significa "no autenticado" (falta o es inválido el token), y `403 Forbidden` significa "autenticado, pero no autorizado" (el usuario es quien dice ser, pero no tiene permiso para esa acción). Es un error clásico de entrevista devolver `403` cuando en realidad no hay token — ese caso es `401`.

## 8. Dónde guardar el token en el cliente

| Storage | Accesible por JS | Riesgo principal |
|---|---|---|
| `localStorage` | Sí | **XSS**: cualquier script inyectado puede leer el token y robarlo |
| Cookie `httpOnly` | No | **CSRF**: se manda automáticamente en cada request a ese dominio, hay que mitigar con `SameSite` y/o un CSRF token |

No hay una opción "segura por default" — es un trade-off: `localStorage` expone el token a XSS, la cookie `httpOnly` lo protege de XSS pero abre la puerta a CSRF si no se configura bien (`SameSite=Strict/Lax` reduce mucho ese riesgo en la práctica).

---
Relacionado: [Autenticación en FastAPI](../python/autenticacion-fastapi.md) (implementación concreta), [Atributos de calidad de sistemas](atributos-de-calidad.md) (escalabilidad stateless), [Idempotencia y tolerancia a fallos](atributos-de-calidad.md).
