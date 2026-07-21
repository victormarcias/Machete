# Endpoints para Microservicios (FastAPI)

Cómo se ve en código Python/FastAPI la construcción de un endpoint pensado para vivir en un microservicio. No es teoría de REST (eso ya está cubierto en otros machetes) — es la sintaxis y los patrones concretos del framework.

## 1. Anatomía de un endpoint en FastAPI

Un endpoint es una función async decorada con el verbo HTTP y el path. `response_model` valida y serializa la respuesta según ese schema, **filtrando cualquier campo del objeto interno que no debería exponerse** (ej. `password_hash`) aunque el objeto real lo tenga.

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class OrderOut(BaseModel):
    id: int
    total: float
    status: str

@app.get("/orders/{order_id}", response_model=OrderOut)
async def get_order(order_id: int):
    order = await order_service.get_by_id(order_id)
    return order  # aunque order tenga más campos internos, solo se serializan los de OrderOut
```

## 2. Path params vs Query params vs Body

FastAPI infiere de dónde viene cada valor según la firma de la función: si el nombre coincide con el path, es path param; un tipo simple que no está en el path es query param; un modelo Pydantic completo es el body. Poner límites (`le`, `ge`, `gt`) en la validación evita que alguien pida `limit=1000000` y tire abajo la base de datos.

```python
from fastapi import Query, Path

@app.get("/orders")
async def list_orders(
    status: str | None = Query(None, description="Filtrar por estado"),
    limit: int = Query(20, le=100),   # ✅ tope máximo, no confiar en que el cliente sea razonable
    offset: int = Query(0, ge=0),
):
    ...

@app.get("/orders/{order_id}")
async def get_order(order_id: int = Path(..., gt=0)):  # ✅ rechaza IDs inválidos antes de tocar la DB
    ...

class CreateOrderBody(BaseModel):
    items: list[str]
    user_id: int

@app.post("/orders")
async def create_order(body: CreateOrderBody):  # un modelo Pydantic como parámetro = viene del body
    ...
```

## 3. Validación de entrada con Pydantic

Pydantic valida tipos, rangos y formatos automáticamente y devuelve un `422` con el detalle exacto de qué campo falló, sin escribir un `if` a mano por cada regla. Los `field_validator` cubren reglas de negocio que un tipo por sí solo no puede expresar.

```python
from pydantic import BaseModel, field_validator

class CreateOrderBody(BaseModel):
    items: list[str]
    user_id: int

    @field_validator("items")
    @classmethod
    def items_no_vacio(cls, v):
        if not v:
            raise ValueError("La orden necesita al menos un item")
        return v

# un request con items=[] devuelve 422 con el mensaje del validator —
# el código del endpoint ni se llega a ejecutar
```

## 4. Dependency Injection con `Depends`

`Depends` resuelve una dependencia compartida (sesión de DB, usuario autenticado desde el token) antes de correr el handler, evitando repetir esa lógica en cada endpoint — es Dependency Inversion (ver [SOLID](../system-design/solid.md)) aplicado a nivel framework: el endpoint recibe una sesión ya lista, sin saber cómo se construyó.

```python
from fastapi import Depends, HTTPException, Header

async def get_db_session():
    session = SessionLocal()
    try:
        yield session       # se inyecta en el endpoint
    finally:
        session.close()      # se cierra solo al terminar el request, sin importar el resultado

async def get_current_user(authorization: str = Header(...)):
    user = decode_token(authorization)
    if not user:
        raise HTTPException(status_code=401, detail="Token inválido")
    return user

@app.get("/orders/{order_id}")
async def get_order(
    order_id: int,
    db=Depends(get_db_session),
    current_user=Depends(get_current_user),
):
    return await order_repository.find(db, order_id, current_user.id)
```

## 5. Manejo de excepciones con `@app.exception_handler`

Centraliza el manejo de errores en un solo lugar en vez de un `try/except` repetido en cada endpoint, y evita el riesgo de seguridad de que un error no manejado devuelva un `500` con el stack trace completo filtrado al cliente. El endpoint solo lanza la excepción de dominio — no sabe nada de códigos HTTP.

```python
from fastapi import Request
from fastapi.responses import JSONResponse

class OrderNotFoundError(Exception):
    def __init__(self, order_id: int):
        self.order_id = order_id

@app.exception_handler(OrderNotFoundError)
async def order_not_found_handler(request: Request, exc: OrderNotFoundError):
    return JSONResponse(status_code=404, content={"error": "order_not_found", "order_id": exc.order_id})

@app.get("/orders/{order_id}")
async def get_order(order_id: int):
    order = await order_service.get_by_id(order_id)
    if not order:
        raise OrderNotFoundError(order_id)
    return order
```

## 6. Documentación automática (OpenAPI)

FastAPI genera Swagger UI (`/docs`) y ReDoc (`/redoc`) automáticamente a partir de los type hints y los modelos Pydantic — no hay un archivo de spec separado que se pueda desincronizar del código real. En microservicios esto importa porque otro equipo necesita conocer el contrato de tu servicio sin leer el código ni pedirte una reunión.

```python
@app.get("/orders/{order_id}", response_model=OrderOut, summary="Obtener un pedido por ID")
async def get_order(order_id: int):
    ...
# con esto, /docs ya muestra el schema completo, los tipos, y ejemplos de request/response
```

## 7. Idempotencia en escrituras

Si el cliente reintenta el mismo `POST` tras un timeout (sin saber si el primer intento llegó a procesarse), sin una idempotency key ese reintento duplica el pedido. Guardando el resultado por key, el segundo intento devuelve la misma respuesta en vez de crear un recurso nuevo.

```python
from fastapi import Header

processed_keys: dict[str, dict] = {}  # en producción: Redis con TTL, no un dict en memoria del proceso

@app.post("/orders")
async def create_order(body: CreateOrderBody, idempotency_key: str = Header(...)):
    if idempotency_key in processed_keys:
        return processed_keys[idempotency_key]  # ya se procesó, no crear un pedido nuevo

    order = await order_service.create(body)
    processed_keys[idempotency_key] = order
    return order
```

## 8. Testing de endpoints con `TestClient`

`TestClient` (basado en `httpx`) permite testear los endpoints sin levantar un servidor real — corre en proceso, así que los tests son rápidos y se pueden correr en CI sin infraestructura extra.

```python
from fastapi.testclient import TestClient

client = TestClient(app)

def test_get_order_not_found():
    response = client.get("/orders/999")
    assert response.status_code == 404
    assert response.json()["error"] == "order_not_found"

def test_create_order_validates_empty_items():
    response = client.post("/orders", json={"items": [], "user_id": 1})
    assert response.status_code == 422  # falla en Pydantic, ni siquiera llega al handler
```

---
Relacionado: [SOLID principles](../system-design/solid.md), [Atributos de calidad de sistemas](../system-design/atributos-de-calidad.md) (idempotencia, tolerancia a fallos), [Sintaxis general](sintaxis.md), [Sync vs Async en FastAPI](sync-vs-async-fastapi.md) (cuándo el `async def` de estos ejemplos realmente aporta algo).
