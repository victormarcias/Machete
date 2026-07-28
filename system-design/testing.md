# Testing — Conceptos Generales

Fundamentos de testing que aplican en cualquier lenguaje. La implementación concreta con pytest/FastAPI está en [Testing en FastAPI](../python/testing-fastapi.md).

## 1. Test pyramid

Los tests se organizan en capas según cuánto cubren y cuánto cuestan: **unit tests** (muchos, rápidos, prueban una función/clase aislada), **integration tests** (menos, prueban varios componentes juntos — ej. que un endpoint realmente persista en una DB de test), y **e2e tests** (pocos, prueban el sistema completo a través de la interfaz real, los más lentos y frágiles).

```
        /\
       /e2e\        pocos, lentos, mucha confianza, se rompen seguido por motivos ajenos al bug
      /------\
     /integr. \     algunos, prueban que las piezas encajan entre sí
    /----------\
   /   unit     \   muchos, rápidos, aislados — la base de la pirámide
  /--------------\
```

Invertir la pirámide (pocos unit tests, muchos e2e — el anti-patrón "ice cream cone") da un test suite lento y flaky: cada bug tarda minutos en confirmarse en vez de milisegundos, y un fallo en la red o el browser rompe tests que no tienen nada que ver con el bug real.

## 2. Test doubles — Mock vs Stub vs Fake vs Spy

Se usan como sinónimos y no lo son — cada uno resuelve un problema distinto al reemplazar una dependencia real en un test:

| Tipo | Qué hace | Verifica el "cómo" fue llamado |
|---|---|---|
| **Stub** | Devuelve una respuesta fija, sin lógica real | No |
| **Mock** | Como un stub, pero además registra y permite verificar las llamadas que recibió | Sí |
| **Fake** | Una implementación real pero simplificada (ej. una DB en memoria en vez de Postgres) | No |
| **Spy** | Envuelve un objeto real, delega la llamada real y además la registra | Sí |

```python
# Stub: devuelve un valor fijo, no le importa cómo ni cuántas veces se lo llamó
class StubPaymentGateway:
    def charge(self, amount, card):
        return {"status": "succeeded"}

# Mock: además del valor de retorno, permite verificar el "cómo" —
# esto es lo que distingue un mock de un simple stub
mock_gateway = Mock()
mock_gateway.charge.return_value = {"status": "succeeded"}
checkout(mock_gateway, cart)
mock_gateway.charge.assert_called_once_with(100, "4242-...")  # ✅ verifica la llamada, no solo el resultado
```

## 3. Por qué mockear servicios externos

Un test que le pega a un servicio externo real (email, pasarela de pago, S3) es **lento** (latencia de red real), **no determinístico** (el servicio puede estar caído, rate-limitado, o devolver algo distinto cada vez) y puede tener **efectos secundarios reales** (mandar un email de verdad, cobrar una tarjeta de verdad, subir un archivo real a un bucket que cuesta dinero). Mockearlo aísla el test del mundo exterior: corre en milisegundos, siempre de la misma forma, sin necesitar credenciales reales en CI.

## 4. Aislamiento de tests

Un test no debería depender del orden en que corre, ni dejar estado que afecte al siguiente. La señal más clara de que algo está mal: los tests pasan corridos individualmente pero fallan al correr todos juntos (o en otro orden) — eso significa que hay estado compartido (una variable global, una fila que quedó en la DB) que un test está heredando de otro sin declararlo explícitamente como dependencia.

## 5. Transactional rollback pattern para tests de DB

Cada test corre dentro de una transacción que se abre antes y se revierte (`ROLLBACK`) al final, en vez de comitear. El test puede insertar, actualizar y borrar libremente — nada de eso queda persistido, así que el siguiente test arranca siempre del mismo estado limpio, sin tener que truncar tablas ni recrear la base entre tests.

```python
# patrón conceptual, independiente del framework
def run_test_in_transaction(test_fn):
    transaction = db.begin()
    try:
        test_fn()
    finally:
        transaction.rollback()  # nada de lo que hizo el test queda persistido
```

Esto es lo que hace que un test suite con cientos de tests de integración contra una DB real siga corriendo en segundos en vez de minutos.

---
Relacionado: [Testing en FastAPI](../python/testing-fastapi.md), [ACID / transacciones / isolation levels](../sql/acid-transacciones-isolation.md) (transacciones y rollback), [Idempotencia](atributos-de-calidad.md).
