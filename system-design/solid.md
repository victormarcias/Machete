# SOLID Principles

Cinco principios de diseño orientado a objetos para escribir código mantenible y fácil de extender sin romper lo existente.

## S — Single Responsibility Principle

Una clase/módulo debería tener una sola razón para cambiar. Si mezclás lógica de negocio, persistencia y notificaciones en una misma clase, un cambio en cómo se envían emails obliga a tocar (y potencialmente romper) la clase que calcula totales.

```python
# ❌ Order mezcla cálculo, persistencia y notificación — 3 razones para cambiar
class Order:
    def calculate_total(self): ...
    def save_to_db(self): ...
    def send_confirmation_email(self): ...

# ✅ cada clase tiene una sola responsabilidad
class Order:
    def calculate_total(self): ...

class OrderRepository:
    def save(self, order: Order): ...

class OrderNotifier:
    def send_confirmation(self, order: Order): ...
```

```ts
// ❌
class Order {
  calculateTotal() { /* ... */ }
  saveToDb() { /* ... */ }
  sendConfirmationEmail() { /* ... */ }
}

// ✅
class Order { calculateTotal() { /* ... */ } }
class OrderRepository { save(order: Order) { /* ... */ } }
class OrderNotifier { sendConfirmation(order: Order) { /* ... */ } }
```

## O — Open/Closed Principle

El código debería estar abierto a extensión pero cerrado a modificación: agregar un caso nuevo no debería requerir tocar código que ya funciona y ya está testeado. Un `if/elif` que crece con cada tipo nuevo viola esto — cada agregado es una oportunidad de romper los casos anteriores.

```python
# ❌ agregar un tipo de descuento nuevo obliga a modificar esta función
def calculate_discount(order, type):
    if type == "vip": return order.total * 0.2
    elif type == "regular": return order.total * 0.1

# ✅ un descuento nuevo = una clase nueva, sin tocar las existentes
class VipDiscount:
    def apply(self, total: float) -> float: return total * 0.2

class RegularDiscount:
    def apply(self, total: float) -> float: return total * 0.1
```

```ts
// ❌
function calculateDiscount(order: Order, type: string) {
  if (type === 'vip') return order.total * 0.2;
  if (type === 'regular') return order.total * 0.1;
}

// ✅
interface DiscountStrategy { apply(total: number): number; }
class VipDiscount implements DiscountStrategy { apply(total: number) { return total * 0.2; } }
class RegularDiscount implements DiscountStrategy { apply(total: number) { return total * 0.1; } }
```

## L — Liskov Substitution Principle

Un subtipo tiene que poder reemplazar a su tipo base sin romper el comportamiento que el código que lo usa espera. El ejemplo clásico: `Square extends Rectangle` parece razonable geométricamente, pero si `Rectangle` expone `setWidth`/`setHeight` de forma independiente, `Square` rompe esa garantía (cambiar el ancho también cambia el alto) — cualquier función que asuma el contrato de `Rectangle` falla al recibir un `Square`.

```python
# ❌ Square rompe el contrato de Rectangle: setWidth también cambia height
class Rectangle:
    def set_width(self, w): self.width = w
    def set_height(self, h): self.height = h

class Square(Rectangle):
    def set_width(self, w): self.width = self.height = w
    def set_height(self, h): self.width = self.height = h
    # test que asume Rectangle: set_width(5); set_height(4) → area debería ser 20, con Square da 16

# ✅ no forzar herencia donde el subtipo no cumple el contrato del padre
class Rectangle:
    def __init__(self, w, h): self.width, self.height = w, h
    def area(self): return self.width * self.height

class Square:
    def __init__(self, side): self.side = side
    def area(self): return self.side ** 2
```

```ts
// ❌
class Rectangle {
  setWidth(w: number) { this.width = w; }
  setHeight(h: number) { this.height = h; }
}
class Square extends Rectangle {
  setWidth(w: number) { this.width = this.height = w; }
  setHeight(h: number) { this.width = this.height = h; }
}

// ✅
interface Shape { area(): number; }
class Rectangle implements Shape {
  constructor(private width: number, private height: number) {}
  area() { return this.width * this.height; }
}
class Square implements Shape {
  constructor(private side: number) {}
  area() { return this.side ** 2; }
}
```

## I — Interface Segregation Principle

No forzar a un cliente a depender de métodos que no usa. Una interfaz "gorda" con muchos métodos obliga a implementaciones parciales a lanzar errores o dejar métodos vacíos — mejor varias interfaces chicas y específicas.

```python
# ❌ interfaz gorda: una impresora simple se ve obligada a "implementar" scan/fax
class MultiFunctionDevice(Protocol):
    def print(self, doc): ...
    def scan(self, doc): ...
    def fax(self, doc): ...

class SimplePrinter(MultiFunctionDevice):
    def print(self, doc): ...
    def scan(self, doc): raise NotImplementedError
    def fax(self, doc): raise NotImplementedError

# ✅ interfaces chicas, cada cliente implementa solo lo que necesita
class Printer(Protocol):
    def print(self, doc): ...

class Scanner(Protocol):
    def scan(self, doc): ...

class SimplePrinter(Printer):
    def print(self, doc): ...
```

```ts
// ❌
interface MultiFunctionDevice {
  print(doc: Doc): void;
  scan(doc: Doc): void;
  fax(doc: Doc): void;
}
class SimplePrinter implements MultiFunctionDevice {
  print(doc: Doc) { /* ... */ }
  scan(doc: Doc): never { throw new Error('not supported'); }
  fax(doc: Doc): never { throw new Error('not supported'); }
}

// ✅
interface Printer { print(doc: Doc): void; }
interface Scanner { scan(doc: Doc): void; }
class SimplePrinter implements Printer { print(doc: Doc) { /* ... */ } }
```

## D — Dependency Inversion Principle

Los módulos de alto nivel (lógica de negocio) no deberían depender de módulos de bajo nivel (detalles de implementación como una DB específica) — ambos deberían depender de una abstracción. Esto es lo que hace posible testear `OrderService` con un repositorio falso, o cambiar de Postgres a Mongo sin tocar la lógica de negocio.

```python
# ❌ OrderService depende directo de una implementación concreta (Postgres)
class PostgresOrderRepository:
    def save(self, order): ...

class OrderService:
    def __init__(self):
        self.repo = PostgresOrderRepository()  # acoplado a Postgres

# ✅ OrderService depende de una abstracción, inyectada desde afuera
class OrderRepository(Protocol):
    def save(self, order): ...

class OrderService:
    def __init__(self, repo: OrderRepository):
        self.repo = repo  # cualquier implementación sirve (Postgres, Mongo, in-memory para tests)
```

```ts
// ❌
class PostgresOrderRepository { save(order: Order) { /* ... */ } }
class OrderService {
  private repo = new PostgresOrderRepository(); // acoplado
}

// ✅
interface OrderRepository { save(order: Order): void; }
class OrderService {
  constructor(private repo: OrderRepository) {} // inyectado
}
```

## Resumen

| Letra | Principio | En una frase |
|---|---|---|
| S | Single Responsibility | Una clase, una razón para cambiar |
| O | Open/Closed | Extender sin modificar código existente |
| L | Liskov Substitution | Un subtipo no debe romper el contrato del tipo base |
| I | Interface Segregation | Interfaces chicas y específicas, no una gorda |
| D | Dependency Inversion | Depender de abstracciones, no de implementaciones concretas |

Relacionado: Controller / Service / Repository (DIP es la base de esa separación) y Patrón Adapter — ver [system-design](README.md).
