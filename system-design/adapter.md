# Patrón Adapter

Patrón estructural que convierte la interfaz de una clase en otra interfaz que el cliente espera, permitiendo que clases con interfaces incompatibles trabajen juntas sin modificar ninguna de las dos.

## Cuándo usarlo

- Integrás una librería de terceros (o una API externa) cuya interfaz no coincide con la que tu código espera, y no podés (o no querés) modificar esa librería.
- Migrás de un proveedor a otro (ej. de Stripe a MercadoPago) y querés que el resto de la app siga hablando con la misma interfaz interna, sin importar el proveedor de turno.
- Necesitás que código legacy con una interfaz vieja conviva con código nuevo escrito contra una interfaz distinta.

## Ejemplo — adaptar un proveedor de pagos externo

El código de la app espera una interfaz propia (`PaymentProcessor`), pero el SDK del proveedor externo tiene su propia forma de nombrar cosas. El adapter traduce entre ambas.

```python
# SDK externo, con su propia interfaz — no lo controlamos
class StripeSDK:
    def create_charge(self, amount_cents: int, currency: str, token: str) -> dict:
        ...  # llama a la API real de Stripe

# Interfaz que espera el resto de la app
class PaymentProcessor(Protocol):
    def pay(self, amount: float, card_token: str) -> bool: ...

# ✅ Adapter: traduce la interfaz de Stripe a la interfaz interna
class StripeAdapter:
    def __init__(self, stripe_sdk: StripeSDK):
        self._sdk = stripe_sdk

    def pay(self, amount: float, card_token: str) -> bool:
        result = self._sdk.create_charge(
            amount_cents=int(amount * 100), currency="usd", token=card_token,
        )
        return result["status"] == "succeeded"

# el resto de la app solo conoce PaymentProcessor, nunca StripeSDK directamente
def checkout(processor: PaymentProcessor, total: float, token: str):
    if processor.pay(total, token):
        print("Pago aprobado")
```

```ts
// SDK externo, con su propia interfaz — no lo controlamos
class StripeSDK {
  createCharge(amountCents: number, currency: string, token: string): { status: string } {
    // llama a la API real de Stripe
    return { status: 'succeeded' };
  }
}

// Interfaz que espera el resto de la app
interface PaymentProcessor {
  pay(amount: number, cardToken: string): boolean;
}

// ✅ Adapter: traduce la interfaz de Stripe a la interfaz interna
class StripeAdapter implements PaymentProcessor {
  constructor(private sdk: StripeSDK) {}

  pay(amount: number, cardToken: string): boolean {
    const result = this.sdk.createCharge(Math.round(amount * 100), 'usd', cardToken);
    return result.status === 'succeeded';
  }
}

// el resto de la app solo conoce PaymentProcessor, nunca StripeSDK directamente
function checkout(processor: PaymentProcessor, total: number, token: string) {
  if (processor.pay(total, token)) console.log('Pago aprobado');
}
```

**El beneficio se ve al cambiar de proveedor**: mañana aparece `MercadoPagoAdapter implements PaymentProcessor` con su propia traducción interna, y `checkout()` no cambia una sola línea.

## Object Adapter vs Class Adapter

- **Object Adapter** (el de los ejemplos de arriba): el adapter **compone** (tiene una referencia a) el objeto que adapta. Funciona en cualquier lenguaje, es el más común en Python/TypeScript/JS.
- **Class Adapter**: el adapter **hereda** de la clase a adaptar además de implementar la interfaz esperada. Requiere herencia múltiple (posible en Python, no en TypeScript/Java/C#) — por eso en el ecosistema JS/TS casi siempre se usa el Object Adapter.

## Relación con otros principios

Un Adapter casi siempre implementa una abstracción (interfaz) que el código de alto nivel define — es una aplicación directa de **Dependency Inversion** (ver [SOLID principles](solid.md)): el código cliente depende de `PaymentProcessor` (la abstracción), no de `StripeSDK` (el detalle concreto).

## Analogía iOS

Es el mismo rol que cumple un `protocol` de Swift envolviendo un SDK de terceros con su propia API (o el viejo patrón de bridging Objective-C ↔ Swift): tu código habla contra el protocolo que vos definiste, y el wrapper se encarga de traducir hacia/desde la librería externa. Si mañana cambiás de SDK, solo se reescribe el wrapper.
