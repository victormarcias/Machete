# Controller / Service / Repository

Patrón de capas para separar responsabilidades en el backend: cada capa sabe hacer una sola cosa, y no le importa cómo funcionan las otras.

## Las tres capas

- **Controller**: recibe el request HTTP, valida/parsea el input, llama al Service, arma la response. No tiene lógica de negocio — es la traducción entre "HTTP" y "la lógica de la app".
- **Service**: la lógica de negocio en sí (calcular un descuento, validar una regla, orquestar varios repositorios). No sabe nada de HTTP ni de SQL.
- **Repository**: acceso a datos — abstrae de qué motor de DB es, qué ORM se usa, cómo está armada la query. El Service le pide datos, no le importa cómo los consigue.

```python
# Repository: solo sabe hablar con la DB
class OrderRepository:
    def find_by_id(self, order_id):
        return db.query(Order).filter(Order.id == order_id).first()
    def save(self, order):
        db.add(order); db.commit()

# Service: lógica de negocio — no sabe nada de HTTP ni de SQL
class OrderService:
    def __init__(self, repo: OrderRepository):
        self.repo = repo

    def apply_discount(self, order_id, percentage):
        order = self.repo.find_by_id(order_id)
        if order.status != "pending":
            raise ValueError("Solo se puede descontar una orden pendiente")
        order.total *= (1 - percentage / 100)
        self.repo.save(order)
        return order

# Controller: solo HTTP — parsea el request, llama al service, arma la response
@app.post("/orders/{order_id}/discount")
async def apply_discount(order_id: int, body: DiscountBody):
    order = order_service.apply_discount(order_id, body.percentage)
    return {"id": order.id, "total": order.total}
```

## Por qué separar

Cada capa tiene una sola razón para cambiar — es [Single Responsibility](solid.md#s--single-responsibility-principle) aplicado a la arquitectura de una request completa: un cambio en el formato de la API toca solo el Controller, un cambio en la regla de negocio toca solo el Service, un cambio de Postgres a Mongo toca solo el Repository.

## El beneficio real: testear sin HTTP ni DB

El Service recibe el Repository como dependencia en vez de crearlo él mismo — [Dependency Inversion](solid.md#d--dependency-inversion-principle) — así que en un test se le puede pasar un Repository falso en memoria en vez del real, y testear la lógica de negocio sin levantar un servidor HTTP ni una base de datos.

```python
class FakeOrderRepository:
    def __init__(self, orders): self.orders = orders
    def find_by_id(self, order_id): return self.orders[order_id]
    def save(self, order): pass  # no hace falta persistir nada para el test

def test_apply_discount_rejects_non_pending_order():
    fake_repo = FakeOrderRepository({1: Order(id=1, status="shipped", total=100)})
    service = OrderService(fake_repo)
    with pytest.raises(ValueError):
        service.apply_discount(1, 10)
```

---
Relacionado: [SOLID principles](solid.md), [Testing — conceptos generales](testing.md#2-test-doubles--mock-vs-stub-vs-fake-vs-spy) (el `FakeOrderRepository` de arriba es un Fake, no un Mock).
