# Python — Setup clásico en macOS (pip + venv + pyenv)

Este documento cubre el flujo **tradicional** de Python, sin `uv`. Útil para entender qué hace cada herramienta por separado antes de saltar a algo todo-en-uno.

---

## 1. Python del sistema vs Python "real"

macOS trae un Python preinstalado en `/usr/bin/python3`. **No tocar, no instalar paquetes ahí** — lo usa el propio sistema operativo internamente.

Para desarrollo hay que instalar Python aparte. Opciones:

| Método | Path resultante | Cuándo usar |
|---|---|---|
| Instalador `.pkg` de [python.org](https://python.org) | `/Library/Frameworks/Python.framework/Versions/X.X/bin/python3` | Simple, directo |
| `brew install python` | `/opt/homebrew/bin/python3` | Si ya usás Homebrew para todo |
| `pyenv install X.X.X` | `~/.pyenv/versions/X.X.X/bin/python3` | Si necesitás varias versiones simultáneas |

**Verificar instalación:**
```bash
python3 --version
which python3   # confirma qué instalación está activa en el PATH
```

⚠️ Si `which python3` devuelve `/usr/bin/python3`, el PATH está mal configurado (apunta al del sistema en vez del que instalaste).

---

## 2. pyenv — manejo de versiones de Python

**Qué es:** herramienta para instalar y alternar entre múltiples versiones de Python en la misma máquina (similar a `nvm` en Node, o manejar distintas versiones de Xcode con `xcode-select`).

**Instalación:**
```bash
brew install pyenv
```

Agregar a `~/.zshrc`:
```bash
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.zshrc
echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.zshrc
echo 'eval "$(pyenv init -)"' >> ~/.zshrc
source ~/.zshrc
```

**Uso:**
```bash
pyenv install 3.12.4     # instala una versión
pyenv global 3.12.4       # la setea como default global
pyenv local 3.11.0        # la setea solo para la carpeta actual (crea .python-version)
python --version          # confirma la versión activa
```

---

## 3. venv — entornos virtuales

**Qué es:** módulo built-in de Python para crear entornos aislados por proyecto. Sin esto, todos los paquetes se instalan globalmente y proyectos distintos pisan sus propias dependencias (versión de Django del proyecto A rompe al proyecto B, por ejemplo).

**Analogía con iOS:** es como tener un `Podfile`/dependencias por proyecto en vez de librerías instaladas a nivel de sistema.

**Crear y usar un venv:**
```bash
cd mi-proyecto
python3 -m venv venv       # crea la carpeta venv/
source venv/bin/activate    # activa el entorno — el prompt muestra (venv)
```

Con el venv activo, todo lo que instales con `pip` queda aislado a ese proyecto:
```bash
pip install --upgrade pip
pip install fastapi uvicorn
```

**Salir del entorno:**
```bash
deactivate
```

**Congelar dependencias (para que otro pueda reproducir el entorno):**
```bash
pip freeze > requirements.txt
```

**Instalar desde ese archivo (en otra máquina o después de clonar el repo):**
```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

---

## 4. Estructura típica de proyecto

```
mi-proyecto/
├── venv/               # NO se commitea
├── requirements.txt    # SÍ se commitea
├── .gitignore          # debe incluir "venv/"
└── main.py
```

`.gitignore` mínimo:
```
venv/
__pycache__/
*.pyc
.env
```

---

## 5. Checklist de setup completo (de cero, sin uv)

```bash
# 1. Instalar pyenv (opcional, solo si necesitás múltiples versiones)
brew install pyenv
pyenv install 3.12.4
pyenv global 3.12.4

# 2. Crear proyecto
mkdir mi-proyecto && cd mi-proyecto

# 3. Crear y activar entorno virtual
python3 -m venv venv
source venv/bin/activate

# 4. Instalar dependencias
pip install --upgrade pip
pip install fastapi uvicorn

# 5. Congelar versiones
pip freeze > requirements.txt

# 6. Correr código
python main.py

# 7. Abrir en VS Code
code .
```

## Notas

- Este flujo es el "clásico" — funciona en cualquier lado, es lo que vas a ver en la mayoría de tutoriales y proyectos legacy.
- El problema: son **3-4 herramientas separadas** (pyenv + venv + pip, a veces + poetry para manejo de dependencias más serio), cada una con su propia sintaxis y configuración.
- Ver [uv — Setup y flujo de trabajo](setup-uv.md) para la alternativa moderna que unifica todo esto en una sola herramienta.
