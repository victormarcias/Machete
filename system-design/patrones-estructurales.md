# Patrones de Diseño — Estructurales

Patrones que definen cómo componer clases y objetos en estructuras más grandes, sin que la complejidad interna se filtre hacia el código cliente.

## Adapter

Convierte la interfaz de una clase en otra interfaz que el cliente espera, permitiendo que clases con interfaces incompatibles trabajen juntas sin modificar ninguna de las dos.

**Cuándo usarlo**: integrás una librería de terceros cuya interfaz no coincide con la que tu código espera y no podés modificarla; migrás de un proveedor a otro (ej. Stripe → MercadoPago) y querés que el resto de la app siga hablando con la misma interfaz interna; o código legacy con una interfaz vieja necesita convivir con código nuevo.

```python
# SDK externo, con su propia interfaz — no lo controlamos
class StripeSDK:
    def create_charge(self, amount_cents: int, currency: str, token: str) -> dict: ...

# Interfaz que espera el resto de la app
class PaymentProcessor(Protocol):
    def pay(self, amount: float, card_token: str) -> bool: ...

# ✅ Adapter: traduce la interfaz de Stripe a la interfaz interna
class StripeAdapter:
    def __init__(self, stripe_sdk: StripeSDK):
        self._sdk = stripe_sdk

    def pay(self, amount: float, card_token: str) -> bool:
        result = self._sdk.create_charge(amount_cents=int(amount * 100), currency="usd", token=card_token)
        return result["status"] == "succeeded"

# el resto de la app solo conoce PaymentProcessor, nunca StripeSDK directamente
def checkout(processor: PaymentProcessor, total: float, token: str):
    if processor.pay(total, token):
        print("Pago aprobado")
```

```ts
class StripeSDK {
  createCharge(amountCents: number, currency: string, token: string) {
    return { status: 'succeeded' }; // llama a la API real de Stripe
  }
}

interface PaymentProcessor { pay(amount: number, cardToken: string): boolean; }

// ✅ Adapter: traduce la interfaz de Stripe a la interfaz interna
class StripeAdapter implements PaymentProcessor {
  constructor(private sdk: StripeSDK) {}
  pay(amount: number, cardToken: string): boolean {
    const result = this.sdk.createCharge(Math.round(amount * 100), 'usd', cardToken);
    return result.status === 'succeeded';
  }
}
```

El beneficio se ve al cambiar de proveedor: mañana aparece `MercadoPagoAdapter implements PaymentProcessor` con su propia traducción interna, y el código que llama `checkout()` no cambia una sola línea. **Object Adapter** (compone el objeto adaptado, como arriba) es el más común en Python/TS/JS; **Class Adapter** (hereda de la clase adaptada) requiere herencia múltiple, posible en Python pero no en TypeScript.

## Decorator

Agrega comportamiento a un objeto en runtime, envolviéndolo, sin modificar su clase ni la de sus hermanos. Los middlewares de Express/Koa son, en esencia, decorators encadenados sobre el request/response.

```python
import time

def with_logging(fn):
    def wrapper(*args, **kwargs):
        print(f"Llamando a {fn.__name__}")
        return fn(*args, **kwargs)
    return wrapper

def with_timing(fn):
    def wrapper(*args, **kwargs):
        start = time.time()
        result = fn(*args, **kwargs)
        print(f"{fn.__name__} tardó {time.time() - start:.3f}s")
        return result
    return wrapper

@with_logging
@with_timing
def process_order(order_id): ...
# cada decorator agrega comportamiento sin tocar el código de process_order
```

```ts
// Express: cada middleware es un decorator que envuelve al siguiente eslabón
app.use((req, res, next) => { console.log(`${req.method} ${req.url}`); next(); }); // logging
app.use((req, res, next) => { req.startTime = Date.now(); next(); }); // timing
app.get('/orders/:id', getOrderHandler); // el handler real no sabe que lo están envolviendo
```

## Facade

Una interfaz simple sobre un subsistema complejo, para que el cliente no tenga que conocer ni orquestar todas las piezas internas.

```python
# ❌ el código cliente conoce y orquesta 3 servicios internos
def checkout(cart, user, card):
    inventory_service.reserve(cart.items)
    payment_service.charge(user, card, cart.total)
    shipping_service.schedule(user.address, cart.items)

# ✅ Facade: el cliente solo conoce una interfaz simple
class CheckoutFacade:
    def complete(self, cart, user, card):
        inventory_service.reserve(cart.items)
        payment_service.charge(user, card, cart.total)
        shipping_service.schedule(user.address, cart.items)

CheckoutFacade().complete(cart, user, card)
```

```ts
class CheckoutFacade {
  complete(cart: Cart, user: User, card: Card) {
    inventoryService.reserve(cart.items);
    paymentService.charge(user, card, cart.total);
    shippingService.schedule(user.address, cart.items);
  }
}

new CheckoutFacade().complete(cart, user, card);
```

## Proxy

Un objeto que controla el acceso a otro, interceptando las llamadas para agregar lógica extra (lazy loading, cache, control de permisos) sin que el cliente note la diferencia — implementa la misma interfaz que el objeto real.

```ts
interface UserRepository { findById(id: string): Promise<User>; }

class RealUserRepository implements UserRepository {
  async findById(id: string) { return db.users.findById(id); }
}

// ✅ Proxy: agrega cache delante del repositorio real, transparente para el cliente
class CachedUserRepository implements UserRepository {
  private cache = new Map<string, User>();
  constructor(private real: UserRepository) {}

  async findById(id: string) {
    if (this.cache.has(id)) return this.cache.get(id)!;
    const user = await this.real.findById(id);
    this.cache.set(id, user);
    return user;
  }
}

const repo: UserRepository = new CachedUserRepository(new RealUserRepository());
```

## Analogía iOS

Adapter y Proxy son el mismo rol que cumple un `protocol` de Swift envolviendo un SDK de terceros, o el viejo bridging Objective-C ↔ Swift: tu código habla contra el protocolo que vos definiste, y el wrapper traduce hacia/desde la librería externa.

---
Relacionado: [SOLID principles](solid.md) (Adapter y Proxy son aplicaciones directas de Dependency Inversion) · catálogo completo en [refactoring.guru](https://refactoring.guru/design-patterns/structural-patterns).
