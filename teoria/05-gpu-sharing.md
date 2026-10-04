# 05 — Compartir GPU: Time-Slicing, MPS y MIG

**Fuentes:** sesiones del 1, 23 y 31 de agosto de 2026

## Time-Slicing
- Comparte la GPU **en el tiempo**, con cambio de contexto entre procesos.
- **Sin aislamiento de memoria ni QoS.**
- Funciona en GPUs de consumo (es software).
- El device plugin anuncia `replicas` GPUs virtuales: Kubernetes cuenta recursos que no existen.

**Comprobado en [Lab 2](../labs/lab-02-k3s-gpu/):** con `replicas: 4`, Kubernetes admitió dos pods de
2.2 GB en una GPU de 4 GB; el segundo murió por OOM y veía el proceso del primero.

## MPS — Multi-Process Service
- Un **daemon es dueño de un único contexto CUDA** y canaliza los kernels de todos los clientes.
- Elimina el cambio de contexto.
- Permite que kernels de distintos procesos ocupen SMs **al mismo tiempo**: compartición **espacial**,
  no temporal.
> TODO: completar con el experimento de MPS en el ConfigMap del device plugin.

## MIG — Multi-Instance GPU
- Partición **física** de SMs, L2 y memoria (ej. `1g.10gb` ≈ 16 SMs de una H100 con su porción de L2 y HBM).
- Aislamiento real: cada instancia tiene su VRAM garantizada.
- Particiones **estáticas**: los SMs que una instancia no usa no los aprovecha otra.
- Solo GPUs de datacenter (A100/H100); la 1650 SUPER no lo soporta.
- Argumento de seguridad poco documentado: la persistencia de memoria después de terminar el pod.

## Comparación
| | Time-Slicing | MPS | MIG |
|---|---|---|---|
| Forma de compartir | Tiempo | Espacio (SMs concurrentes) | Partición física |
| Aislamiento de VRAM | No | No | Sí |
| Cambio de contexto | Sí | No | No aplica |
| Flexibilidad | Alta | Alta | Baja (estático) |
| Hardware | Cualquiera | Cualquiera compatible | Datacenter |

## Combinación
MIG, MPS y Time-Slicing se pueden **apilar** (por ejemplo, MPS o Time-Slicing dentro de una instancia MIG).

## Conexión con el objetivo
La estrategia de sharing decide cuántas réplicas de un modelo caben en una GPU y con qué riesgo: más
densidad con Time-Slicing/MPS, más garantías con MIG.
