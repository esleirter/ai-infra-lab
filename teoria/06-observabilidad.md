# 06 — Observabilidad de GPU

**Fuentes:** sesiones del 1, 4 y 23 de agosto de 2026

## GPU-Util (nvidia-smi / `DCGM_FI_DEV_GPU_UTIL`)
Porcentaje del tiempo de muestreo en que hubo **algún kernel activo**. Es un booleano promediado, no un
porcentaje de capacidad: un kernel usando pocos SMs marca 100% igual que la saturación.

- Durante la generación con TinyLlama marcaba ~98%: el overhead de Python entre `generate()` deja huecos.
- Con 2.50 GB de VRAM ocupada puede marcar 0%: **utilización y memoria son ejes independientes**.

## Métricas reales de profiling (DCGM)
| Métrica | Qué mide |
|---|---|
| `DCGM_FI_PROF_SM_ACTIVE` | Fracción de SMs con trabajo |
| `DCGM_FI_PROF_SM_OCCUPANCY` | Ocupación de warps en los SMs |
| `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` | Si de verdad se usan los Tensor Cores |
| `DCGM_FI_PROF_DRAM_ACTIVE` | Actividad de la memoria |

**Solo en GPUs del segmento datacenter.** Las GeForce no las exponen aunque la arquitectura lo permita
(restricción de segmento, no de generación). Requieren `--cap-add SYS_ADMIN`.

## Proxies en GeForce
| Métrica | Uso |
|---|---|
| `DCGM_FI_DEV_POWER_USAGE` | Proxy honesto de cómputo real |
| `DCGM_FI_DEV_SM_CLOCK` | Sube bajo carga real |
| `DCGM_FI_DEV_MEM_COPY_UTIL` | Lo más cercano a `DRAM_ACTIVE` |
| `DCGM_FI_DEV_FB_USED` / `FB_FREE` | VRAM usada / libre |
| `DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION` | Base para costo por token |

Medido en [Lab 3](../labs/lab-03-dcgm/): 20.2 W en reposo vs 40.7 W bajo carga; SM clock 1.20 → 1.92 GHz.

## DCGM y dcgm-exporter
- **DCGM (Data Center GPU Manager):** herramienta de NVIDIA para gestionar y monitorear GPUs.
- **dcgm-exporter:** expone las métricas en formato Prometheus.
- **Atribución por pod:** montando `/var/lib/kubelet/pod-resources` y con
  `DCGM_EXPORTER_KUBERNETES=true`, consulta al kubelet qué pod tiene qué GPU.
- El DaemonSet **no debe pedir `nvidia.com/gpu`**.

## Intervalo de scrape
Con 30 s las cargas cortas de inferencia se promedian a nada; en el lab se usó 5 s.

## Conexión con el objetivo
Sin distinguir utilización real, memoria y energía, no se puede saber si una GPU está saturada o solo
ocupada, ni calcular el costo por token.
