# Testing en FastAPI (pytest)

Implementación concreta con `pytest`, `httpx.AsyncClient` y mocking de servicios externos. Los conceptos de fondo (test pyramid, test doubles, por qué mockear, aislamiento) están en [Testing — conceptos generales](../system-design/testing.md).

## 1. `pytest` fixtures y `conftest.py`

Una fixture es una función de setup/teardown reutilizable. `conftest.py` guarda las fixtures compartidas por todo el proyecto — pytest las descubre solas, sin que ningún test tenga que importarlas. Todo lo que sigue al `yield` corre como teardown, siempre, incluso si el test falla.

```python
# conftest.py
import pytest

@pytest.fixture
def sample_user():
    return {"email": "ana@mail.com", "password": "secret123"}

@pytest.fixture
def db_session():
    session = SessionLocal()
    yield session       # el test usa la sesión acá
    session.close()      # teardown: corre siempre, haya pasado o fallado el test
```

Cualquier test de cualquier archivo puede pedir `sample_user` o `db_session` como parámetro de la función — pytest los resuelve automáticamente.

## 2. Transactional rollback en SQLAlchemy async

Aplicando el patrón de [Testing — conceptos generales](../system-design/testing.md) con una fixture real: se abre una transacción sobre la conexión, se la usa para crear la sesión que ve el test, y se revierte al final.

```python
@pytest.fixture
async def db_session():
    async with engine.connect() as conn:
        trans = await conn.begin()
        session = AsyncSession(bind=conn)
        yield session          # el test hace sus inserts/updates acá
        await trans.rollback()  # se revierte todo — la DB queda exactamente como estaba
```

Cada test arranca de un estado limpio sin truncar tablas ni recrear la base, y corre rápido porque nunca llega a comprometer nada en disco.

## 3. `httpx.AsyncClient` para testear rutas async

`TestClient` (visto en [Endpoints para microservicios](endpoints-microservicios.md)) funciona para rutas sync y async porque maneja su propio event loop por dentro, pero cuando toda la app usa fixtures async (como la sesión de DB del punto 2), `AsyncClient` encaja mejor: es awaitable y se integra directo con `pytest-asyncio` sin loops anidados.

```python
import pytest
from httpx import AsyncClient, ASGITransport

@pytest.mark.asyncio
async def test_get_order():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as client:
        response = await client.get("/orders/1")
    assert response.status_code == 200
```

## 4. Mockear servicios externos

`pytest-mock` para funciones propias (como el envío de email); `moto` para simular servicios de AWS completos (como S3) sin pegarle a la nube real.

```python
# ✅ mockear el envío de mail: el test no manda ningún email real
def test_register_sends_welcome_email(mocker):
    mock_send = mocker.patch("app.services.email.send_welcome_email")
    client.post("/register", json={"email": "ana@mail.com", "password": "secret123"})
    mock_send.assert_called_once_with("ana@mail.com")

# ✅ moto: simula S3 en memoria, sin pegarle a AWS real ni gastar nada
from moto import mock_aws
import boto3

@mock_aws
def test_upload_to_s3():
    s3 = boto3.client("s3", region_name="us-east-1")
    s3.create_bucket(Bucket="mi-bucket-test")
    upload_file(s3, "mi-bucket-test", "foto.jpg", b"contenido")
    objects = s3.list_objects(Bucket="mi-bucket-test")
    assert objects["Contents"][0]["Key"] == "foto.jpg"
```

## 5. Testear rutas protegidas por auth

Reutilizando el mismo helper que usa `/login` en producción (ver [Autenticación en FastAPI](autenticacion-fastapi.md)) para generar un token válido dentro de una fixture, en vez de loguearse de verdad en cada test.

```python
@pytest.fixture
def auth_headers():
    token = create_access_token(user_id=1)  # mismo helper que /login en producción
    return {"Authorization": f"Bearer {token}"}

def test_get_me(client, auth_headers):
    response = client.get("/me", headers=auth_headers)
    assert response.status_code == 200
```

## 6. Testear file uploads

```python
def test_upload_avatar(client, auth_headers):
    file_content = b"fake-image-bytes"
    response = client.post(
        "/users/me/avatar",
        headers=auth_headers,
        files={"file": ("avatar.jpg", file_content, "image/jpeg")},
    )
    assert response.status_code == 200
```

## 7. Testear background tasks

En producción, una `BackgroundTask` de FastAPI corre después de mandar la respuesta. En los tests, como `TestClient`/`AsyncClient` ejecutan todo en el mismo proceso, la tarea ya terminó para cuando la aserción se ejecuta — no hace falta esperar ni hacer polling, solo mockear la función y verificar que se llamó.

```python
def test_create_order_triggers_notification(client, auth_headers, mocker):
    mock_notify = mocker.patch("app.services.notifications.notify_warehouse")
    response = client.post("/orders", headers=auth_headers, json={"items": ["sku-1"]})
    assert response.status_code == 201
    mock_notify.assert_called_once()  # la background task ya corrió para cuando llega acá
```

## 8. Testear ownership/authorization checks

El caso clásico: un usuario autenticado intenta tocar un recurso que no le pertenece — la respuesta correcta es `403`, no `401` (ver [Autenticación vs Autorización](../system-design/autenticacion.md)).

```python
def test_cannot_delete_other_users_order(client, auth_headers_user_a, order_owned_by_user_b):
    response = client.delete(f"/orders/{order_owned_by_user_b.id}", headers=auth_headers_user_a)
    assert response.status_code == 403  # autenticado, pero no autorizado para este recurso
```

---
Relacionado: [Testing — conceptos generales](../system-design/testing.md), [Endpoints para microservicios](endpoints-microservicios.md), [Autenticación en FastAPI](autenticacion-fastapi.md), [Sync vs Async en FastAPI](sync-vs-async-fastapi.md).
