# Lab 3 — DCGM + Prometheus + Grafana

**Fecha:** 23 de agosto de 2026 · **Teoría relacionada:** [06 — Observabilidad](../../teoria/06-observabilidad.md)

## Objetivo
Observabilidad de GPU con atribución por pod y comprobar empíricamente por qué `GPU-Util` engaña.

## 1. Verificación previa: ¿qué métricas expone esta GPU?
Antes de montar nada, correr el exporter suelto:
```bash
docker run --rm --gpus all --cap-add SYS_ADMIN -p 9400:9400 \
  nvcr.io/nvidia/k8s/dcgm-exporter:4.2.3-4.1.3-ubuntu22.04

curl -s localhost:9400/metrics | grep -E "^DCGM_FI_PROF"   # vacío
```
**Cero métricas `DCGM_FI_PROF_*`** (sin `SM_ACTIVE`, `SM_OCCUPANCY`, `DRAM_ACTIVE`). Es restricción de
**segmento de producto** (GeForce vs datacenter), no de arquitectura: la TU116 es compute capability 7.5.
`--cap-add SYS_ADMIN` es obligatorio para el módulo de profiling incluso en hardware que lo soporta.

### Métricas disponibles en la GTX 1650 SUPER
| Métrica | Uso |
|---|---|
| `DCGM_FI_DEV_GPU_UTIL` | El sospechoso a desmentir |
| `DCGM_FI_DEV_POWER_USAGE` | **Proxy honesto de cómputo real** (cap 100 W) |
| `DCGM_FI_DEV_SM_CLOCK` | Sube bajo carga real |
| `DCGM_FI_DEV_MEM_COPY_UTIL` | Lo más cercano a `DRAM_ACTIVE` |
| `DCGM_FI_DEV_FB_USED` / `FB_FREE` | VRAM: el recurso que Time-Slicing no gobierna |
| `DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION` | Base para calcular costo por token |

El lab cambió de eje, no de valor: en vez de *GPU-Util vs SM occupancy*, quedó **GPU-Util vs watts**.

## 2. Despliegue
| Componente | Detalle |
|---|---|
| dcgm-exporter | DaemonSet en `kube-system`, imagen `nvcr.io/nvidia/k8s/dcgm-exporter:4.2.3-4.1.3-ubuntu22.04` |
| kube-prometheus-stack | Helm, release `prom`, namespace `monitoring` |
| Grafana | NodePort 30300 |
| Dashboard | NVIDIA oficial, Grafana.com ID **12239** |

### Configuración clave
- **Atribución por pod:** el exporter monta `/var/lib/kubelet/pod-resources` (solo lectura) y usa
  `DCGM_EXPORTER_KUBERNETES=true` para preguntarle al kubelet qué pod tiene qué GPU.
- **El DaemonSet no pide `nvidia.com/gpu`**: necesita ver todas las GPUs, no que le asignen una. Si la
  pidiera, consumiría una de las 4 réplicas de Time-Slicing.
- **ServiceMonitor con `interval: 5s`:** con el default de 30 s, las cargas cortas de inferencia se
  promedian a nada.

Manifiestos: [`manifests/dcgm-exporter.yaml`](manifests/dcgm-exporter.yaml) (DaemonSet + Service),
[`manifests/dcgm-servicemonitor.yaml`](manifests/dcgm-servicemonitor.yaml) y
[`manifests/values-kube-prometheus-stack.yaml`](manifests/values-kube-prometheus-stack.yaml) (values del
release `prom`: Grafana NodePort 30300 y `serviceMonitorSelectorNilUsesHelmValues: false` para que
Prometheus recoja ServiceMonitors ajenos al chart).

## 3. Experimento: utilización vs memoria
- `hog-1`: reserva 2.20 GB y duerme.
- `burn`: bucle sostenido de multiplicación de matrices FP16.

| Pod | VRAM | GPU-Util | Potencia | SM clock |
|---|---|---|---|---|
| hog-1 | 2.50 GB | 0% | — | — |
| burn | 2.71 GB | 100% | **40.7 W** | 1.92 GHz |
| Reposo (memoria ocupada, sin trabajo) | — | — | 20.2 W | 1.20 GHz |

Manifiesto: [`manifests/burn.yaml`](manifests/burn.yaml) (matmul 4096×4096 FP16 en bucle con
`torch.cuda.synchronize()`).

> TODO: agregar capturas del dashboard.

## 4. Desvío de debugging: Grafana mentía
Parecía que las métricas no llegaban a Prometheus. La causa era el **caché del autocompletado de
Grafana**. Se resolvió consultando la API de Prometheus directamente:
```bash
curl -s http://<prometheus>/api/v1/label/__name__/values | tr ',' '\n' | grep -i dcgm
curl -s 'http://<prometheus>/api/v1/query?query=DCGM_FI_DEV_POWER_USAGE'
```

## Lecciones
1. Utilización y memoria son ejes independientes y ambos hacen falta: 2.50 GB ocupados con 0% util.
   GPU-Util es un booleano promediado (¿hubo algún kernel?), no un porcentaje de capacidad.
2. Sin métricas de profiling, la potencia es el proxy honesto de cómputo (20.2 W → 40.7 W).
3. Las `DCGM_FI_PROF_*` son de segmento datacenter: planificar la observabilidad según el segmento,
   no la generación.
4. Ante un dato que "no aparece", consultar la API de la fuente antes que la UI.

## Pendiente para Lab 5 (vLLM)
Ver `GPU-Util` alto con potencia oscilante durante decode memory-bound.
