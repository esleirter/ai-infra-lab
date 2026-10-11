# 05 — Compartir GPU: Time-Slicing, MPS y MIG

**Fuentes:** sesiones del 1, 23 y 31 de agosto y del 11 de octubre de 2026

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

**Comprobado en [Lab 4](../labs/lab-04-helm-mps/)** (device plugin por Helm, `sharing.mps` con `replicas: 2`):
- Requiere el DaemonSet `mps-control-daemon` y la label `nvidia.com/mps.capable=true` (la pone GFD);
  el manifiesto crudo del plugin no trae ninguno de los dos.
- La GPU pasa a `Compute M.` = `E. Process`: el único proceso C es `nvidia-cuda-mps-server` (28 MiB) y
  los pods aparecen como `M+C`. CUDA fuera de Kubernetes probablemente falle por el modo exclusivo.
- **Tope de memoria por cliente** = nominal / réplicas (4096 / 2 = 2048 MiB), menos ~42 MiB de contexto
  del cliente. Es fijo por réplica configurada, no por pods corriendo: la memoria de la réplica vacía no
  se presta. Más aislamiento = menos aprovechamiento.
- **El tope es límite máximo, no reserva garantizada:** el cliente recibe el mínimo entre lo que le
  queda de su tope y la memoria física libre. Como los topes se calculan sobre memoria nominal y el
  driver reserva ~380 MiB (CUDA ve 3.63 GiB), la suma de topes puede exceder lo usable y el segundo pod
  hace OOM dentro de su tope. La garantía existe solo si la suma de topes cabe en la memoria usable:
  **calcular réplicas sobre memoria usable, no nominal.**
- Costo fijo: servidor 28 MiB una vez + ~42 MiB por cliente (vs ~54 MiB de contexto por pod con
  Time-Slicing).

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
| Aislamiento de VRAM | No | Tope por cliente (máximo, no garantía) | Sí |
| Cambio de contexto | Sí | No | No aplica |
| Flexibilidad | Alta | Media (topes fijos por réplica) | Baja (estático) |
| Hardware | Cualquiera | Cualquiera compatible | Datacenter |

## Combinación
MIG, MPS y Time-Slicing se pueden **apilar** (por ejemplo, MPS o Time-Slicing dentro de una instancia MIG).

## Conexión con el objetivo
La estrategia de sharing decide cuántas réplicas de un modelo caben en una GPU y con qué riesgo: más
densidad con Time-Slicing/MPS, más garantías con MIG.
