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

## 6. Hooks de setup/teardown — `beforeEach`, `afterEach`, `beforeAll`, `afterAll`

La mayoría de los frameworks de testing ofrecen cuatro momentos distintos para código que corre **alrededor** de los tests, no como parte de ellos (se sobreentiende que estos nombres son de JS — Jest/Mocha/Vitest —, pero el concepto se repite en pytest, JUnit, RSpec, etc. con otra sintaxis):

- **`beforeEach`**: corre antes de **cada** test del bloque — el lugar típico para resetear estado (crear un objeto fresco, limpiar una DB de test).
- **`afterEach`**: corre después de **cada** test — cleanup puntual.
- **`beforeAll`**: corre **una sola vez**, antes del primer test del bloque — para setup costoso y compartible (levantar un servidor de test, abrir una conexión).
- **`afterAll`**: corre **una sola vez**, después del último test — cleanup final de lo que se abrió al principio.

```python
# equivalente conceptual en Python — la sintaxis exacta varía según el framework
def before_all():       # equivalente a beforeAll — una sola vez, antes de todos los tests del bloque
    return connect_test_db()

def after_all(db):       # equivalente a afterAll — una sola vez, al final
    db.close()

def before_each(db):     # equivalente a beforeEach — antes de CADA test, clave para el aislamiento
    db.clear()

def after_each():        # equivalente a afterEach — después de cada test
    reset_mocks()
```

**Por qué importa el "una vez" vs "cada vez"**: usar `beforeAll` para algo que debería resetearse por test rompe el [aislamiento de tests](#4-aislamiento-de-tests) del punto 4 — si un test muta un estado que `beforeAll` solo creó una vez, el siguiente test hereda esa mutación sin declararlo como dependencia. Regla práctica: `beforeAll`/`afterAll` para trabajo caro y realmente compartible; `beforeEach`/`afterEach` para todo lo que necesita empezar limpio en cada test.

pytest no tiene estas cuatro funciones como tal — resuelve lo mismo con el parámetro `scope` de una fixture (ver la implementación completa en [Testing en FastAPI](../python/testing-fastapi.md#9-hooks-de-setupteardown-en-pytest-scope-de-las-fixtures)).

## 7. Tests frágiles vs tests robustos

No es una dicotomía formal con nombre propio como "Mock vs Stub" — es más una consecuencia directa de **qué** se testea. Un test frágil (*brittle*) verifica **detalles de implementación**: se rompe con cualquier refactor interno, aunque el comportamiento externo siga siendo correcto. Un test robusto (*resilient*) verifica **comportamiento observable** (el resultado, el contrato) — sobrevive a refactors porque no le importa *cómo* se llegó al resultado, solo *que* sea el correcto.

```python
# ❌ frágil: testea implementación — se rompe si cambia el orden interno de llamadas,
# aunque el resultado final (el descuento aplicado) siga siendo correcto
def test_apply_discount_calls_repo_in_order(mocker):
    repo = mocker.Mock()
    service = OrderService(repo)
    service.apply_discount(1, 10)
    assert repo.method_calls == [call.find_by_id(1), call.save(ANY)]  # detalle interno, no comportamiento

# ✅ robusto: testea comportamiento — no le importa cómo se calculó el resultado
def test_apply_discount_reduces_total():
    repo = FakeOrderRepository({1: Order(id=1, status="pending", total=100)})
    service = OrderService(repo)
    order = service.apply_discount(1, 10)
    assert order.total == 90  # el resultado observable, sin importar el camino interno
```

Es la máxima de "testear comportamiento, no implementación" (popularizada por Kent C. Dodds): si un refactor que no cambia ningún resultado observable rompe un test, ese test estaba acoplado a la implementación, no al contrato.

## 8. Caja negra vs caja blanca

- **Caja negra (black box)**: testear el comportamiento externo sin mirar ni conocer la implementación interna — solo importa qué entra y qué sale, según el contrato/spec. El test se podría escribir sin haber leído una línea del código.
- **Caja blanca (white box)**: testear conociendo la implementación interna — los casos se diseñan mirando el código: cada rama de un `if`, cada loop, cada camino posible, buscando cobertura de código completa.

```python
def calculate_shipping(weight, country):
    if country == "AR":
        return weight * 2 if weight > 10 else 10
    return weight * 5

# ✅ caja negra: los casos salen del contrato/spec, sin mirar el código —
# "envío a Argentina cuesta esto, a otro país cuesta esto otro"
def test_shipping_ar_light():
    assert calculate_shipping(5, "AR") == 10

def test_shipping_other_country():
    assert calculate_shipping(5, "US") == 25

# ✅ caja blanca: los casos salen de mirar las ramas del código —
# hace falta ver el if/else para saber que weight > 10 en AR es un camino distinto
def test_shipping_ar_heavy_branch():
    assert calculate_shipping(15, "AR") == 30  # cubre la rama que la caja negra de arriba se salteó
```

**Trade-off**: caja negra es más robusta a refactors (no le importa cómo está hecho — ver el punto 7) y refleja mejor el contrato que le importa al usuario real. Caja blanca encuentra casos borde que el spec no contempló explícitamente (esa rama `weight > 10` capaz nadie la pidió, pero existe en el código y hay que cubrirla) — es la base de las métricas de *code coverage*. En la práctica, la mayoría de los tests de un equipo son caja negra a nivel de comportamiento, con caja blanca usada puntualmente para cazar ramas sin cubrir.

## 9. Fixtures

Un **fixture** es el estado conocido y reproducible desde el que arranca un test — datos de prueba, un objeto ya construido, o el entorno ya preparado (una conexión, un usuario de prueba). La idea es que el test nunca dependa de "lo que haya quedado" de una corrida anterior: arranca siempre del mismo punto de partida, declarado explícitamente.

Es el mecanismo concreto detrás de dos cosas ya vistas en este archivo:
- Resuelve el [aislamiento de tests](#4-aislamiento-de-tests) del punto 4 — cada test recibe su propio estado fresco, en vez de heredar el de otro test.
- Es una forma de implementar los [hooks de setup/teardown](#6-hooks-de-setupteardown--beforeeach-aftereach-beforeall-afterall) del punto 6 — en frameworks como pytest, un fixture directamente **reemplaza** a esos hooks (según su `scope`), en vez de coexistir como un mecanismo aparte.

```python
# fixture como dato: un objeto de prueba ya armado, listo para usar
fixture_user = {"id": 1, "email": "test@mail.com", "role": "admin"}

# fixture como función que provee ese estado, con su propio setup/teardown
def user_fixture():
    user = create_test_user()
    try:
        yield user
    finally:
        delete_test_user(user.id)
```

No todos los frameworks lo modelan igual: en pytest es una **función inyectable** (con setup/teardown propio vía `yield`); en muchos frameworks de JS es más común que sea **datos estáticos** (un archivo JSON/objeto de ejemplo) cargados al principio del test. La idea de fondo — estado conocido y reproducible — es la misma en los dos casos.

Ver la implementación en pytest en [Testing en FastAPI](../python/testing-fastapi.md#1-pytest-fixtures-y-conftestpy).

## 10. Ejecución de tests en paralelo vs serial

Muchos test runners son capaces de repartir los tests entre varios workers/procesos para correr en **paralelo** y aprovechar los cores de la máquina — en algunos esto es opt-in (hay que pedirlo explícitamente), en otros es el default. Forzar lo contrario — correr todo **secuencialmente, en un solo proceso**, uno atrás del otro — sirve en dos casos típicos:

- **Debuggear un test flaky** (un test que a veces pasa y a veces falla, **sin que el código haya cambiado** — la señal de que hay algo no determinístico de por medio, no un bug real en la lógica): si sospechás que es por [aislamiento de tests](#4-aislamiento-de-tests) roto (dos tests paralelos pisándose un recurso compartido — un puerto, un archivo, una fila de DB), correr serial aísla la variable: si el test deja de fallar, confirma que el problema era paralelismo/estado compartido, no el test en sí.
- **CI con recursos limitados**: en un runner con pocos cores o poca memoria, levantar N workers en paralelo puede terminar siendo más lento (por contención) que correr todo serial, o directamente quedarse sin memoria.

```python
# pytest: serial es el default — el paralelismo es opt-in con el plugin pytest-xdist
pytest              # serial, un test atrás del otro
pytest -n auto      # paralelo, reparte entre workers — esto es lo que se "apaga" para volver a serial
```

Cada ecosistema nombra esto distinto (en Jest, JS, es el flag `--runInBand`) pero el concepto es el mismo en cualquier lenguaje: forzar un solo proceso para eliminar el paralelismo como variable.

**La causa raíz real**: si correr serial "arregla" un test que fallaba en paralelo, el problema no es el paralelismo en sí — es que ese test no estaba realmente aislado, compartía estado con otro. Forzar serial es un diagnóstico/workaround para confirmar la sospecha, no la solución: el fix real es garantizar que cada test tenga su propio [fixture](#9-fixtures) sin nada compartido con los demás.

---
Relacionado: [Testing en FastAPI](../python/testing-fastapi.md), [ACID / transacciones / isolation levels](../sql/acid-transacciones-isolation.md) (transacciones y rollback), [Idempotencia](atributos-de-calidad.md).
