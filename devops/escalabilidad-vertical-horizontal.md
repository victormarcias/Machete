# Escalabilidad vertical vs horizontal

Las dos formas de darle más capacidad a un sistema.

**Vertical**: agregarle más CPU/RAM a la misma máquina. Simple — no cambia nada de la arquitectura — pero tiene techo físico (no existe una instancia infinitamente grande) y casi siempre implica downtime al redimensionar (hay que rebootear).

**Horizontal**: agregar más instancias corriendo en paralelo. Sin techo teórico, pero exige que la app sea *stateless* (sesión en un store compartido, no en memoria del proceso — ver el ejemplo de [Escalabilidad](../system-design/atributos-de-calidad.md#escalabilidad)) y necesita un load balancer repartiendo el tráfico entre instancias.

| | Vertical | Horizontal |
|---|---|---|
| Cómo se hace | Más CPU/RAM a la misma máquina | Más instancias en paralelo |
| Techo | Físico (el tamaño máximo de instancia que exista) | Teóricamente ninguno |
| Downtime al escalar | Sí, casi siempre (reboot) | No, se suma una instancia más sin tocar las que ya corren |
| Requisito previo | Ninguno | App stateless + load balancer |

En la práctica se combinan: cada instancia de un cluster horizontal también tiene un tamaño (vertical) elegido a su vez.

---
Relacionado: [Escalabilidad](../system-design/atributos-de-calidad.md#escalabilidad), [Diagnóstico DevOps](../diagnostico/devops.md).
