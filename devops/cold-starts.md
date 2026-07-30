# Cold starts

Latencia extra que paga la primera request cuando una función serverless o un contenedor que [escala a cero](vps-vs-cloud-run.md) tiene que arrancar desde cero: inicializar el runtime, cargar dependencias, y a veces abrir conexiones (DB, etc.) antes de poder responder. Se nota como picos de latencia intermitentes en tráfico esporádico — las requests siguientes, mientras la instancia sigue "caliente", no pagan ese costo.

**Mitigaciones típicas**:
- *Provisioned concurrency* — pagar por mantener N instancias siempre calientes, sacrificando parte del ahorro de escalar a cero.
- Reducir el tamaño del paquete/imagen y las dependencias que se cargan al arrancar.
- Abrir conexiones (DB, clientes HTTP) fuera del handler, para que se reutilicen entre invocaciones calientes en vez de recrearse en cada cold start — ver el ejemplo en [Diagnóstico DevOps](../diagnostico/devops.md).

---
Relacionado: [Diagnóstico DevOps](../diagnostico/devops.md), [Deploy a Cloud Run](deploy-cloud-run.md) (escala a cero), [Elasticidad](../system-design/atributos-de-calidad.md#elasticidad).
