# Arquitectura Kafka

Apache Kafka es una plataforma de streaming distribuido — la pieza central para desacoplar servicios que necesitan comunicarse por eventos en vez de llamadas directas.

## Qué es Kafka

A diferencia de una cola de mensajes tradicional (RabbitMQ, SQS), Kafka es un **log distribuido append-only**: los mensajes se escriben secuencialmente y se conservan durante un tiempo/tamaño configurable — **no se borran al ser consumidos**. Cualquier consumer puede leer desde cualquier punto del log, incluso mensajes que otro consumer ya leyó hace rato.

## Topics y particiones

Un **topic** es un canal con nombre al que se publican mensajes (ej. `order_events`). Por dentro, cada topic se divide en **particiones** — el log real es por partición, no por topic. Kafka garantiza orden **dentro** de una partición, pero **no entre particiones distintas** del mismo topic: si el orden entre dos mensajes importa, tienen que ir a la misma partición (típicamente eligiendo la partición según una key, para que todos los eventos relacionados queden juntos y ordenados).

```python
# el mismo order_id siempre va a la misma partición → sus eventos quedan ordenados entre sí
producer.send('order_events', key=str(order_id), value=event_data)
```

## Producers / Consumers y Consumer Groups

Los **producers** publican mensajes a un topic; los **consumers** los leen. Varios consumers pueden agruparse en un **consumer group**: Kafka reparte las particiones del topic entre los consumers de ese grupo, así que cada mensaje lo procesa **un solo consumer del grupo** — es la forma de escalar el consumo horizontalmente, agregando consumers al grupo hasta un máximo de un consumer por partición.

```python
consumer = KafkaConsumer('order_events', group_id='inventory-service')
# si hay 4 particiones y 2 consumers en el grupo, cada consumer procesa 2 particiones
```

Si dos consumer groups distintos leen el mismo topic (ej. `inventory-service` y `analytics-service`), **cada grupo** recibe una copia completa de todos los mensajes — son independientes entre sí.

## Offset

Cada consumer trackea su posición en cada partición con un **offset** (un número que avanza secuencialmente). Como los mensajes no se borran al leerse, un consumer puede reiniciar desde el último offset guardado si se cae (no pierde mensajes), o rebobinar el offset a mano para reprocesar mensajes viejos (ej. reconstruir un estado tras un bug). Esto es lo que habilita **replay** — algo que una cola tradicional no ofrece, porque ahí un mensaje desaparece apenas se hace `ack`.

## Kafka vs cola tradicional

| | Cola tradicional (RabbitMQ, SQS) | Kafka |
|---|---|---|
| Mensaje tras ser consumido | Se borra | Se conserva (según retención configurada) |
| Múltiples consumers del mismo mensaje | Requiere fan-out explícito | Cada consumer group lee todo, independiente |
| Replay | No | Sí (rebobinar el offset) |
| Orden garantizado | Por cola | Por partición |

---
Relacionado: [Idempotencia](atributos-de-calidad.md#idempotencia) (un consumer puede reprocesar el mismo mensaje más de una vez tras un crash — el handler necesita ser idempotente), [Observabilidad](atributos-de-calidad.md#observabilidad).
