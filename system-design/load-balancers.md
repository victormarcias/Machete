# Load Balancers

Reparten el tráfico entrante entre varias instancias de un servicio — la pieza que hace posible la [escalabilidad horizontal](atributos-de-calidad.md#escalabilidad).

## L4 vs L7

- **L4 (transport layer)**: decide a dónde mandar el tráfico mirando solo IP y puerto — no abre el contenido del paquete. Más rápido (menos trabajo por request), pero no puede rutear según el contenido de la request.
- **L7 (application layer)**: entiende el protocolo de aplicación (HTTP) — puede rutear según path (`/api` a un servicio, `/admin` a otro), header, o cookie. Más lento que L4 (tiene que parsear el request), pero mucho más flexible.

```
# L4: decide solo por IP:puerto — no sabe qué path está pidiendo el cliente
443 → instancia A o B (round robin, sin mirar la request)

# L7: puede rutear por contenido
/api/*    → cluster de backend
/static/* → cluster de assets estáticos
```

## Algoritmos de balanceo

- **Round robin**: reparte en orden circular, uno por uno. Simple, funciona bien si todas las instancias tienen capacidad similar.
- **Least connections**: manda la siguiente request a la instancia con menos conexiones activas en este momento — mejor cuando las requests tienen duración muy variable (una instancia con requests lentas no sigue recibiendo más tráfico solo porque "le tocaba" por turno).
- **Sticky sessions (IP hash)**: la misma IP de cliente siempre va a la misma instancia — útil si hay estado en memoria del lado del servidor que no se puede compartir (lo ideal, ver [Escalabilidad](atributos-de-calidad.md#escalabilidad), es no necesitar esto).

## Quién es quién

| Nombre | Capa | Notas |
|---|---|---|
| **NLB** (AWS Network Load Balancer) | L4 | Máximo throughput, mínima latencia |
| **ALB** (AWS Application Load Balancer) | L7 | Ruteo por path/header, integra con auth |
| **Nginx** | L7 (también sirve como L4) | El más usado como reverse proxy propio — ver [Deploy a un VPS](../devops/deploy-vps.md) |
| **HAProxy** | L4 y L7 | Especializado en balanceo, muy usado delante de bases de datos también |
| **Traefik** | L7 | Se integra nativo con Docker/K8s, detecta servicios nuevos automáticamente |
| **Envoy** | L7 | Proxy de datos usado como base de service meshes (Istio) |
| **K8s Ingress** | L7 | La forma estándar de exponer servicios HTTP de un cluster K8s hacia afuera |

---
Relacionado: [Kubernetes](../devops/kubernetes.md), [Escalabilidad](atributos-de-calidad.md#escalabilidad), [Deploy a un VPS](../devops/deploy-vps.md) (Nginx como reverse proxy).
