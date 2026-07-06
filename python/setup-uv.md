# uv — Setup y flujo de trabajo en macOS

Este documento asume que ya tenés Python instalado (ver [Setup clásico en macOS](setup-macos.md)). Acá el foco es **uv**, la alternativa moderna que reemplaza pip + venv + pyenv + poetry.

---

## Qué es uv y qué problema resuelve

Flujo clásico de Python = **varias herramientas separadas**:

| Herramienta | Para qué servía |
|---|---|
| `venv` | Crear entornos virtuales (aislar dependencias por proyecto) |
| `pip` | Instalar paquetes |
| `pyenv` | Manejar múltiples versiones de Python |
| `poetry` / `pipenv` | Manejar dependencias del proyecto (`pyproject.toml`, lockfiles) |

Cuatro herramientas, cuatro configuraciones, cuatro fuentes de bugs.

**uv** = gestor de proyectos Python hecho por **Astral** (mismo equipo de `ruff`, el linter). Escrito en Rust.

Reemplaza a **pip + venv + pyenv + poetry** en una sola herramienta:

- **Velocidad**: instala paquetes 10-100x más rápido que pip (paralelismo + cache global)
- **Todo-en-uno**: maneja versión de Python, venv, y dependencias del proyecto
- **Lockfile automático**: reproducibilidad garantizada (`uv.lock`)
- **No hay que activar venv manualmente**: `uv run` lo detecta y usa solo

### Analogía con iOS

Si venís de Swift: `uv` es a Python lo que **Swift Package Manager** es a Xcode — maneja dependencias, versiones y build en un solo lugar, en vez de tener herramientas sueltas.

---

## Instalación

```bash
brew install uv
```

Verificar:
```bash
which uv        # /opt/homebrew/bin/uv
uv --version
```

---

## Flujo de trabajo con uv

### 1. Crear proyecto nuevo

```bash
cd ~/Documents/GitHub
uv init nombre-proyecto
cd nombre-proyecto
```

Fijar versión específica de Python (recomendado si la última es muy reciente/inestable):
```bash
uv init nombre-proyecto --python 3.12
```

**Estructura generada:**
```
nombre-proyecto/
├── .python-version
├── pyproject.toml     # manifiesto del proyecto (deps, config)
├── README.md
└── main.py
```

### 2. Agregar dependencias

```bash
uv add fastapi uvicorn
```

Esto:
- Crea el venv automáticamente (oculto, `.venv/`)
- Instala los paquetes
- Los registra en `pyproject.toml`
- Genera/actualiza `uv.lock` (versiones exactas, para reproducibilidad)

### 3. Correr código

```bash
uv run main.py
```

No hace falta `source .venv/bin/activate` — `uv run` detecta el entorno del proyecto y lo usa automáticamente.

### 4. Dependencias solo de desarrollo

```bash
uv add --dev pytest ruff
```

### 5. Reproducir un proyecto ya existente (clonado del repo)

```bash
git clone <repo>
cd <repo>
uv sync       # instala exactamente lo que dice uv.lock
```

---

## Comparación: flujo clásico vs uv

| Acción | Flujo clásico (pip + venv) | Con uv |
|---|---|---|
| Instalar versión de Python | `pyenv install 3.12.4` | `uv python install 3.12` |
| Crear entorno | `python3 -m venv venv` | *(automático al hacer `uv add`)* |
| Activar entorno | `source venv/bin/activate` | *(no hace falta)* |
| Instalar paquete | `pip install fastapi` | `uv add fastapi` |
| Correr script | `python main.py` (con venv activo) | `uv run main.py` |
| Congelar versiones | `pip freeze > requirements.txt` | *(automático, `uv.lock`)* |
| Reproducir entorno en otra máquina | `pip install -r requirements.txt` | `uv sync` |

---

## Checklist de setup completo con uv

```bash
# 1. Instalar uv
brew install uv

# 2. Crear proyecto
cd ~/Documents/GitHub
uv init mi-proyecto --python 3.12
cd mi-proyecto

# 3. Agregar dependencias
uv add fastapi uvicorn

# 4. Correr
uv run main.py

# 5. Abrir en VS Code
code .
```

## Notas

- `uv.lock` **sí se commitea** (para reproducibilidad), el `.venv/` **no** (va en `.gitignore`)
- Python 3.14 es muy reciente — si una librería tira error de compatibilidad, downgradear a 3.12 con `uv init --python 3.12`
- `uv` también puede instalar y manejar versiones de Python directamente (`uv python install 3.12`), sin necesidad de `pyenv` en paralelo
