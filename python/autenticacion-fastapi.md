# Autenticación en FastAPI

Implementación concreta de registro + login con Argon2 y JWT. Los conceptos de fondo (por qué hashear, qué es un salt, JWT stateless) están en [Autenticación y Seguridad — conceptos generales](../system-design/autenticacion.md).

## 1. Hashing de passwords con Argon2 (`passlib`)

`passlib` genera el salt automáticamente y lo embebe en el string resultante — no hay que manejarlo a mano.

```python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["argon2"], deprecated="auto")

hashed = pwd_context.hash("mi-password-super-secreta")
pwd_context.verify("mi-password-super-secreta", hashed)  # True
pwd_context.verify("password-incorrecta", hashed)          # False
```

## 2. Registro: nunca persistir ni devolver el password en texto plano

El hash se calcula antes de guardar, y la respuesta del endpoint nunca incluye ni el password original ni el hash — filtrar el hash no es tan grave como el plano, pero sigue siendo información que un atacante puede usar para intentar crackearlo offline.

```python
class RegisterBody(BaseModel):
    email: str
    password: str  # nunca se persiste tal cual

@app.post("/register")
async def register(body: RegisterBody, db: AsyncSession = Depends(get_db)):
    hashed_password = pwd_context.hash(body.password)
    user = User(email=body.email, hashed_password=hashed_password)
    db.add(user)
    await db.commit()
    return {"id": user.id, "email": user.email}  # nunca el hash en la respuesta
```

## 3. Login y generación de JWT

El mismo mensaje de error para "usuario no existe" y "password incorrecta" evita que alguien pueda usar el endpoint para descubrir qué emails están registrados (*user enumeration*) — dar mensajes distintos es una filtración de información sutil pero real.

```python
from datetime import datetime, timedelta, timezone
from jose import jwt
from fastapi.security import OAuth2PasswordRequestForm

def create_access_token(user_id: int) -> str:
    expire = datetime.now(timezone.utc) + timedelta(minutes=15)
    payload = {"sub": str(user_id), "exp": expire}
    return jwt.encode(payload, settings.SECRET_KEY, algorithm=settings.ALGORITHM)

@app.post("/login")
async def login(form_data: OAuth2PasswordRequestForm = Depends(), db: AsyncSession = Depends(get_db)):
    user = await get_user_by_email(db, form_data.username)
    # ✅ mismo error tanto si el usuario no existe como si la password está mal
    if not user or not pwd_context.verify(form_data.password, user.hashed_password):
        raise HTTPException(status_code=401, detail="Credenciales inválidas")
    token = create_access_token(user.id)
    return {"access_token": token, "token_type": "bearer"}
```

## 4. `pydantic-settings` para configuración

`SECRET_KEY` hardcodeado en el código termina, tarde o temprano, commiteado a git. Leerlo desde variables de entorno / `.env` (que va en `.gitignore`, ver [Setup clásico](setup-macos.md)) evita que el secreto viaje con el código fuente.

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    SECRET_KEY: str
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 15

    class Config:
        env_file = ".env"

settings = Settings()  # lee de variables de entorno / .env — nunca hardcodeado en el código
```

## 5. Proteger rutas con `OAuth2PasswordBearer` + `Depends`

`OAuth2PasswordBearer` extrae automáticamente el header `Authorization: Bearer <token>`, y hace que el botón "Authorize" aparezca solo en la documentación de `/docs` (ver [Documentación automática](endpoints-microservicios.md)) — cualquier ruta que dependa de `get_current_user` queda protegida sin repetir la lógica de validación en cada endpoint.

```python
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="login")

async def get_current_user(token: str = Depends(oauth2_scheme), db: AsyncSession = Depends(get_db)):
    try:
        payload = jwt.decode(token, settings.SECRET_KEY, algorithms=[settings.ALGORITHM])
        user_id = payload.get("sub")
    except JWTError:
        raise HTTPException(status_code=401, detail="Token inválido")
    user = await get_user_by_id(db, int(user_id))
    if not user:
        raise HTTPException(status_code=401, detail="Usuario no encontrado")
    return user

@app.get("/me")
async def read_current_user(current_user: User = Depends(get_current_user)):
    return {"email": current_user.email}
```

## 6. Qué nunca loguear

Un log de debug con datos sensibles puede terminar meses en un sistema de logging centralizado, expuesto a cualquiera con acceso a esos logs — mucho más gente de la que debería poder ver una contraseña.

```python
# ❌ incluso en debug, esto puede terminar persistido y accesible por terceros
logger.debug(f"Login attempt: {form_data.username} / {form_data.password}")

# ✅ el evento se loguea sin ningún dato sensible
logger.info(f"Login attempt for user_id={user.id if user else 'unknown'}")
```

---
Relacionado: [Autenticación y Seguridad — conceptos generales](../system-design/autenticacion.md), [Endpoints para microservicios](endpoints-microservicios.md) (`Depends`, exception handlers), [Sync vs Async en FastAPI](sync-vs-async-fastapi.md).
