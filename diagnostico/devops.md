# Diagnóstico DevOps

Causas más comunes de lentitud a nivel infraestructura, de más a menos frecuentes.

## Falta de escalabilidad vertical

El servidor se quedó chico para la carga actual (CPU/RAM al límite) y nadie subió el tamaño de instancia. Antes de complejizar con más nodos, confirmar si escalar verticalmente resuelve el cuello de botella actual. Ver [devops](../devops/README.md).

## Mal load balancing en K8s

Pods sin **readiness probes** bien configuradas reciben tráfico antes de estar listos; falta de **HPA** (Horizontal Pod Autoscaler) hace que la carga no se redistribuya al agregar réplicas; `resource limits` mal seteados generan throttling de CPU silencioso. Ver [devops](../devops/README.md).

```yaml
# ✅ readinessProbe: no recibe tráfico hasta responder 200 en /health
readinessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 10
resources:
  requests: { cpu: "250m", memory: "256Mi" }
  limits: { cpu: "500m", memory: "512Mi" }
```

## Red lenta / falta de CDN

Assets estáticos (imágenes, JS, CSS) servidos directo desde el origin obligan a cada usuario a viajar hasta donde sea que esté ese servidor — si el origin está en EE.UU. y el usuario en Argentina, cada request arrastra esa latencia geográfica aunque el archivo nunca cambie. Una CDN cachea copias en servidores (*edge nodes*) cercanos a cada usuario, así el archivo se sirve desde el punto más cercano en vez de cruzar el mundo en cada request.

```
# ✅ le dice a la CDN que puede cachear el asset por 1 año (los archivos con hash en el nombre
# no cambian nunca, así que un cache largo es seguro)
Cache-Control: public, max-age=31536000, immutable
```

## Cold start

Funciones serverless (Lambda, Cloud Functions) o contenedores que escalan a cero cuando no hay tráfico tardan en arrancar la primera request tras estar inactivos: hay que inicializar el runtime, cargar dependencias y a veces establecer conexiones (DB, etc.) antes de responder. Se nota como latencia alta e intermitente en requests esporádicos. Mitigaciones típicas: *provisioned concurrency*, mantener una instancia mínima siempre caliente, o reducir el tamaño del paquete/dependencias para arrancar más rápido. Ver [devops](../devops/README.md).

```js
// ✅ Node Lambda: conexión a DB fuera del handler, se reutiliza entre invocaciones "calientes"
const db = connectToDb(); // corre una sola vez por cold start, no en cada request

export const handler = async (event) => {
  const result = await db.query('SELECT ...');
  return { statusCode: 200, body: JSON.stringify(result) };
};
```
