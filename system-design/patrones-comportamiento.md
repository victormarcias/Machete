# Patrones de Diseño — Comportamiento

Patrones que definen cómo se comunican y reparten responsabilidades los objetos entre sí.

## Strategy

Familia de algoritmos intercambiables, cada uno encapsulado en su propia clase, seleccionable en runtime. Evita los `if/elif` gigantes que crecen con cada caso nuevo y violan Open/Closed.

```python
class VipDiscount:
    def apply(self, total: float) -> float: return total * 0.2

class RegularDiscount:
    def apply(self, total: float) -> float: return total * 0.1

def checkout(total: float, strategy):
    return total - strategy.apply(total)

checkout(100, VipDiscount())  # el algoritmo de descuento se elige desde afuera, sin tocar checkout()
```

Ver el ejemplo completo (con TypeScript) en [Open/Closed Principle](solid.md) — Strategy es la implementación concreta de ese principio.

## Observer

Un objeto (subject) mantiene una lista de dependientes (observers) y les notifica automáticamente cualquier cambio de estado. Es la base de los event emitters, pub/sub, y de cómo React decide re-renderizar cuando cambia el estado.

```python
class EventEmitter:
    def __init__(self):
        self._listeners = {}

    def on(self, event: str, callback):
        self._listeners.setdefault(event, []).append(callback)

    def emit(self, event: str, *args):
        for cb in self._listeners.get(event, []):
            cb(*args)

emitter = EventEmitter()
emitter.on("order_created", lambda order: send_confirmation_email(order))
emitter.on("order_created", lambda order: update_inventory(order))
emitter.emit("order_created", order)  # ambos listeners se disparan, desacoplados entre sí
```

```ts
class EventEmitter {
  private listeners: Record<string, ((...args: any[]) => void)[]> = {};

  on(event: string, callback: (...args: any[]) => void) {
    (this.listeners[event] ??= []).push(callback);
  }

  emit(event: string, ...args: any[]) {
    this.listeners[event]?.forEach((cb) => cb(...args));
  }
}

const emitter = new EventEmitter();
emitter.on('order_created', (order) => sendConfirmationEmail(order));
emitter.emit('order_created', order);
```

## Command

Encapsula una acción (y sus parámetros) como un objeto, en vez de ejecutarla directo. Permite encolar, loguear, reintentar o deshacer esa acción sin acoplar quién la dispara con quién la ejecuta.

```ts
interface Command { execute(): Promise<void>; }

class SendEmailCommand implements Command {
  constructor(private to: string, private body: string) {}
  async execute() { await emailService.send(this.to, this.body); }
}

// ✅ la cola solo necesita saber ejecutar Commands, no conocer cada tipo de tarea
class JobQueue {
  private queue: Command[] = [];
  push(cmd: Command) { this.queue.push(cmd); }
  async processAll() { for (const cmd of this.queue) await cmd.execute(); }
}

const queue = new JobQueue();
queue.push(new SendEmailCommand('ana@mail.com', 'Bienvenida'));
```

## Chain of Responsibility

Pasa una request a través de una cadena de handlers, donde cada uno decide si la procesa, la modifica, o la pasa al siguiente. Los middlewares (Express, Django, cualquier framework HTTP) son el ejemplo más común de este patrón en el día a día.

```python
class Handler:
    def __init__(self, next_handler=None):
        self.next = next_handler

    def handle(self, request):
        if self.next:
            return self.next.handle(request)

class AuthHandler(Handler):
    def handle(self, request):
        if not request.get("token"):
            raise PermissionError("No autenticado")
        return super().handle(request)

class RateLimitHandler(Handler):
    def handle(self, request):
        if is_rate_limited(request["user_id"]):
            raise Exception("Rate limit excedido")
        return super().handle(request)

chain = AuthHandler(RateLimitHandler())
chain.handle(request)  # pasa por auth, luego rate limit, luego el resto de la lógica
```

---
Relacionado: [SOLID principles](solid.md) · catálogo completo en [refactoring.guru](https://refactoring.guru/design-patterns/behavioral-patterns).
