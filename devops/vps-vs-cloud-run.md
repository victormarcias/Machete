# VPS vs Cloud Run

Comparación entre [Deploy a un VPS](deploy-vps.md) (servidor propio, siempre prendido) y [Deploy a Cloud Run](deploy-cloud-run.md) (contenedor serverless, escala a cero).

## Comparación técnica

| | VPS | Cloud Run |
|---|---|---|
| Quién administra el SO | Vos (parches, firewall, SSH) | Google |
| Costo con cero tráfico | Full precio, 24/7 | ~$0 (escala a cero) |
| Cold start | No existe (siempre prendido) | Sí, en la primera request tras estar en cero |
| Certificado HTTPS | Manual (`certbot`, renovación cada 90 días) | Automático |
| Control fino de infraestructura | Total | Limitado a lo que expone la plataforma |

## Pros y contras

| | VPS | Cloud Run |
|---|---|---|
| **Pros** | Costo fijo y predecible con tráfico alto y constante; control total (podés instalar lo que sea, tunear el kernel, correr varios servicios en la misma máquina); sin cold start; sin atarte a un proveedor específico | Cero mantenimiento de SO/parches; escala automáticamente con la demanda (y a cero cuando no hay tráfico); HTTPS y balanceo de carga resueltos por la plataforma; pagás por uso real, no por capacidad reservada |
| **Contras** | Vos sos responsable de seguridad, updates y capacity planning; pagás lo mismo tenga tráfico o no; escalar significa aprovisionar más máquinas a mano (o armar tu propio autoscaling) | Cold start en tráfico esporádico; menos control (no hay acceso al SO subyacente); a tráfico alto y sostenido puede terminar costando más que una máquina propia; atado a las convenciones de la plataforma (límites de tiempo de request, tamaño de imagen, etc.) |

---
Relacionado: [Deploy a un VPS](deploy-vps.md), [Deploy a Cloud Run](deploy-cloud-run.md), [Cold start](../diagnostico/devops.md), [Elasticidad](../system-design/atributos-de-calidad.md#elasticidad).
