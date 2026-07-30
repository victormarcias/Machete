# Kubernetes — readiness probes, HPA, resource limits

Tres piezas básicas de K8s que definen si un cluster escala bien o mal.

## Readiness probe

Un chequeo periódico que le dice a K8s si un pod está listo para recibir tráfico. Sin esto, un pod recién arrancado (todavía inicializando conexiones, cargando config) recibe requests igual y las falla — la readiness probe hace que quede "fuera de rotación" hasta responder OK.

## HPA (Horizontal Pod Autoscaler)

Agrega o quita réplicas de un pod automáticamente según el uso real de recursos (CPU, memoria, o una métrica custom). Sin HPA, escalar significa que alguien tiene que notar la carga y cambiar el número de réplicas a mano.

## Resource limits

`requests` es lo que el pod pide reservado (K8s no lo agenda en un nodo que no pueda garantizarlo); `limits` es el techo — si el pod lo supera, se lo throttlea (CPU) o se lo mata por OOM (memoria). Sin límites, un pod con un memory leak puede consumir toda la memoria del nodo y tirar abajo a sus vecinos.

```yaml
readinessProbe:
  httpGet: { path: /health, port: 3000 }
  periodSeconds: 10
resources:
  requests: { cpu: "250m", memory: "256Mi" }
  limits: { cpu: "500m", memory: "512Mi" }
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
```

Las tres piezas trabajan juntas: HPA decide cuántos pods hacen falta, resource limits define cuánto puede consumir cada uno, y la readiness probe evita que un pod recién creado por el HPA reciba tráfico antes de estar listo.

---
Relacionado: [Disponibilidad](../system-design/atributos-de-calidad.md#disponibilidad), [Elasticidad](../system-design/atributos-de-calidad.md#elasticidad), [Diagnóstico DevOps](../diagnostico/devops.md).
