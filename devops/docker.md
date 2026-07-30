# Dockerización de una app FastAPI

Cómo empaquetar una app en una imagen reproducible, sin importar a dónde se despliegue después (VPS, Cloud Run, K8s). Es el paso previo y compartido a cualquiera de esos destinos.

## 1. Por qué contenedorizar

Sin Docker, "funciona en mi máquina" depende de qué versión de Python tenés instalada, qué paquetes del sistema operativo están presentes, y qué variables de entorno seteaste a mano hace 3 meses y ya no te acordás. Un contenedor empaqueta el runtime, las dependencias y el código en una sola imagen inmutable — lo que corre en tu laptop es *exactamente* lo que corre en producción, byte por byte.

## 2. Multi-stage build

Un build normal instala compiladores y herramientas de build (`gcc`, headers de desarrollo) que la app necesita para *instalarse* pero no para *correr* — y esas herramientas terminan viajando a la imagen final, infladas y con más superficie de ataque. Un multi-stage build usa una imagen para compilar/instalar, y copia solo el resultado final a una imagen limpia — las herramientas de build nunca llegan a producción.

```dockerfile
# ---- Stage 1: build ----
FROM python:3.12-slim AS builder
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen --no-dev

# ---- Stage 2: runtime ----
FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /app/.venv /app/.venv   # ✅ solo el resultado, no las herramientas que lo generaron
COPY . .
ENV PATH="/app/.venv/bin:$PATH"
CMD ["fastapi", "run", "main.py", "--port", "8000"]
```

La imagen final no tiene `uv` ni ningún cache de instalación — solo el venv ya armado y el código.

## 3. `.dockerignore`

Sin esto, `COPY . .` copia **todo** el directorio a la imagen — incluyendo `.git/`, `.venv/` local, `__pycache__/`, y cualquier `.env` con secretos. Además de inflar la imagen, es una forma fácil de filtrar credenciales sin querer.

```
# .dockerignore
.git
.venv
__pycache__
*.pyc
.env
.pytest_cache
```

## 4. Imagen base: `slim` vs completa

`python:3.12` (completa) trae compiladores y librerías de desarrollo que casi nunca hacen falta en runtime. `python:3.12-slim` es una fracción del tamaño, con solo lo esencial para correr Python — el trade-off es que si una dependencia necesita compilar algo nativo (ej. una librería con extensión en C), puede fallar en `slim` por faltarle un header del sistema, y ahí hay que instalarlo explícitamente en el stage de build.

```dockerfile
# si una dependencia necesita compilar algo nativo, agregar lo mínimo necesario en el builder stage:
RUN apt-get update && apt-get install -y --no-install-recommends gcc libpq-dev && rm -rf /var/lib/apt/lists/*
```

## 5. Build y run local

```bash
docker build -t miapp .
docker run -p 8000:8000 --env-file .env miapp
```

`--env-file .env` inyecta variables de entorno sin hardcodearlas en la imagen (ver [pydantic-settings](../python/autenticacion-fastapi.md) para cómo la app las lee) — la imagen en sí no tiene ningún secreto embebido, así que se puede compartir/pushear a un registry sin filtrar nada.

---
Relacionado: [Deploy a Cloud Run](deploy-cloud-run.md), [Deploy a un VPS](deploy-vps.md), [Autenticación en FastAPI](../python/autenticacion-fastapi.md) (`pydantic-settings`).
