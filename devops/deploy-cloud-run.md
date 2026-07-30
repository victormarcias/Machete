# Deploy a Cloud Run (contenedores serverless)

Cómo llevar la imagen de [Dockerización de una app FastAPI](docker.md) a producción sin administrar un servidor propio. Alternativa a [Deploy a un VPS](deploy-vps.md) cuando no querés vos ser responsable de parchear el SO, renovar certificados a mano, ni pagar por una máquina prendida las 24hs aunque nadie la use.

## 1. Qué es un contenedor serverless y qué significa "escala a cero"

En un VPS, la máquina está prendida y facturando todo el tiempo, tenga tráfico o no. Cloud Run corre tu contenedor **solo cuando llega una request** — si no hay tráfico, apaga la instancia por completo (escala a **cero** réplicas) y no pagás nada por ese tiempo. La primera request después de estar en cero paga el costo de arrancar el contenedor de nuevo — ver [Cold start](../diagnostico/devops.md).

```bash
# el mismo comando de build de siempre — Cloud Run no necesita nada especial en el Dockerfile
docker build -t gcr.io/mi-proyecto/miapp .
```

## 2. Serverless Postgres con Neon

Una DB tradicional (RDS, Cloud SQL) sigue el mismo modelo "siempre prendida" que un VPS — no tiene sentido pairearla con un compute que escala a cero, porque la DB seguiría facturando 24/7 aunque el compute esté apagado la mayor parte del tiempo. **Neon** ofrece Postgres que también escala a cero (y reactiva la conexión en el primer query), completando el modelo serverless de punta a punta.

```python
# la app se conecta igual que a cualquier Postgres — Neon es compatible, no hace falta un driver especial
DATABASE_URL = "postgresql+asyncpg://user:pass@ep-xxx.neon.tech/miapp"
```

## 3. Deploy del container

```bash
docker push gcr.io/mi-proyecto/miapp

gcloud run deploy miapp \
  --image gcr.io/mi-proyecto/miapp \
  --platform managed \
  --region us-central1 \
  --set-env-vars DATABASE_URL=$DATABASE_URL \
  --allow-unauthenticated
```

`--set-env-vars` inyecta config igual que el `--env-file` local del [Dockerfile](docker.md) — la imagen sigue sin tener secretos embebidos. `--allow-unauthenticated` expone el servicio públicamente (sin eso, Cloud Run exige un token de IAM en cada request, útil para servicios internos entre microservicios pero no para una API pública).

## 4. Dominio custom + HTTPS

A diferencia del VPS (donde vos instalás Nginx y corrés `certbot` a mano — ver [Let's Encrypt](deploy-vps.md)), Cloud Run gestiona el certificado TLS automáticamente al mapear un dominio: solo hay que apuntar el DNS y Google se encarga de emitir y renovar el certificado.

```bash
gcloud run domain-mappings create --service miapp --domain api.miapp.com --region us-central1
# devuelve los registros DNS (CNAME/A) que hay que agregar en tu proveedor de dominio
```

## 5. Security headers vía middleware

El servidor no agrega por sí solo headers de seguridad — hay que declararlos explícitamente. Sin ellos, la app queda expuesta a ataques que esos headers específicamente mitigan (clickjacking sin `X-Frame-Options`, MIME sniffing sin `X-Content-Type-Options`, downgrade a HTTP sin `Strict-Transport-Security`).

```python
from starlette.middleware.base import BaseHTTPMiddleware

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        response = await call_next(request)
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["Strict-Transport-Security"] = "max-age=63072000; includeSubDomains"
        return response

app.add_middleware(SecurityHeadersMiddleware)
```

## VPS vs Cloud Run

| | VPS | Cloud Run |
|---|---|---|
| Quién administra el SO | Vos (parches, firewall, SSH) | Google |
| Costo con cero tráfico | Full precio, 24/7 | ~$0 (escala a cero) |
| Cold start | No existe (siempre prendido) | Sí, en la primera request tras estar en cero |
| Certificado HTTPS | Manual (`certbot`, renovación cada 90 días) | Automático |
| Control fino de infraestructura | Total | Limitado a lo que expone la plataforma |

### Pros y contras

| | VPS | Cloud Run |
|---|---|---|
| **Pros** | Costo fijo y predecible con tráfico alto y constante; control total (podés instalar lo que sea, tunear el kernel, correr varios servicios en la misma máquina); sin cold start; sin atarte a un proveedor específico | Cero mantenimiento de SO/parches; escala automáticamente con la demanda (y a cero cuando no hay tráfico); HTTPS y balanceo de carga resueltos por la plataforma; pagás por uso real, no por capacidad reservada |
| **Contras** | Vos sos responsable de seguridad, updates y capacity planning; pagás lo mismo tenga tráfico o no; escalar significa aprovisionar más máquinas a mano (o armar tu propio autoscaling) | Cold start en tráfico esporádico; menos control (no hay acceso al SO subyacente); a tráfico alto y sostenido puede terminar costando más que una máquina propia; atado a las convenciones de la plataforma (límites de tiempo de request, tamaño de imagen, etc.) |

---
Relacionado: [Dockerización de una app FastAPI](docker.md), [Deploy a un VPS](deploy-vps.md), [Cold start](../diagnostico/devops.md), [Elasticidad](../system-design/atributos-de-calidad.md#elasticidad).
