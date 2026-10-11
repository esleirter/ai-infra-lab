# Errores encontrados

Cada error con síntoma, causa raíz y solución. Ordenados por lab.

| # | Lab | Síntoma | Causa raíz | Solución |
|---|---|---|---|---|
| 1 | 0 | Se cae la red (RTL8125) tras instalar el driver | Paquete de módulos incompleto en kernel 5.15.0-186 | Arrancar con kernel HWE 6.8.0-136 |
| 2 | 0 | DKMS: `unrecognized option -ftrivial-auto-var-init=zero` | Kernel compilado con GCC 12, GCC activo 11.4 | Instalar `gcc-12` y `update-alternatives` |
| 3 | 0 | `nvidia-ctk runtime configure` falla con EOF | `daemon.json` vacío (0 bytes), JSON inválido | Escribir `{}` y repetir |
| 4 | 2 | Labels del nodo no aplicadas | k3s arrancó antes de existir `config.yaml`; el script no reinicia si el binario no cambia | `sudo systemctl restart k3s` |
| 5 | 2 | Device plugin: `Incompatible strategy detected auto`, `No devices found` | El pod del plugin corría con runc, sin `/dev/nvidia*` | Patch del DaemonSet con `runtimeClassName: nvidia` |
| 6 | 2 | Pods GPU fallan con `No such file or directory` | Faltaba la solicitud o el RuntimeClass; hacen falta ambos | Pod con `nvidia.com/gpu: 1` **y** `runtimeClassName: nvidia` |
| 7 | 2 | Pod GPU: `exec: "nvidia-smi": executable file not found in $PATH` | El runtime inyecta `nvidia-smi` en un hook de prearranque; el `command` se resuelve contra el PATH de la imagen antes | Ruta absoluta `/usr/bin/nvidia-smi` o imagen CUDA |
| 8 | 2 | Device plugin en `CrashLoopBackOff`: `config.yaml: no such file or directory` | ConfigMap con key `any` (convención Helm); el key es el filename montado | Key `config.yaml` |
| 9 | 2 | `hog-2` muere con `CUDA out of memory` | Time-Slicing no aísla VRAM (comportamiento esperado) | Usar MIG si hace falta garantía de memoria |
| 10 | 3 | Métricas DCGM "no llegan" a Grafana | Caché del autocompletado de Grafana | Consultar la API de Prometheus directamente |
| 11 | 3 | Sin métricas `DCGM_FI_PROF_*` | Restricción del segmento GeForce | Usar potencia y SM clock como proxy |
| 12 | 4 | `hog-1` (2.2 GB) OOM con MPS `replicas: 2`: `1.96 GiB is free`; con Time-Slicing funcionaba | Tope por réplica = 4096/2 = 2048 MiB − ~42 MiB de contexto; la réplica vacía no presta memoria | Reservar por debajo del tope por réplica |
| 13 | 4 | `hog-2` (1.9 GB) OOM con `1.36 GiB is free`, dentro de su tope | Topes calculados sobre 4096 MiB nominales; el driver reserva ~380 MiB y el escritorio ~248 MiB. El cliente recibe el mínimo entre su tope y la memoria física libre | Calcular réplicas sobre memoria usable (3.63 GiB), no nominal |
| 14 | 4 | Pods `hog` quedan ~30 s en `Terminating` | `python` es PID 1 y no maneja SIGTERM | `terminationGracePeriodSeconds: 0` en `hog.yaml` |

## Patrones
- **Los mensajes de error despistan:** "did you configure the Container Toolkit?" cuando el toolkit
  estaba bien (#5).
- **Verificar cómo se instaló algo antes de configurarlo** (#8).
- **Ir a la fuente antes que a la UI** (#10).
- **Los límites se calculan sobre valores nominales; la realidad es la memoria usable** (#13).
