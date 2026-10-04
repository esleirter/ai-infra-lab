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
| 7 | 2 | Device plugin en `CrashLoopBackOff`: `config.yaml: no such file or directory` | ConfigMap con key `any` (convención Helm); el key es el filename montado | Key `config.yaml` |
| 8 | 2 | `hog-2` muere con `CUDA out of memory` | Time-Slicing no aísla VRAM (comportamiento esperado) | Usar MIG si hace falta garantía de memoria |
| 9 | 3 | Métricas DCGM "no llegan" a Grafana | Caché del autocompletado de Grafana | Consultar la API de Prometheus directamente |
| 10 | 3 | Sin métricas `DCGM_FI_PROF_*` | Restricción del segmento GeForce | Usar potencia y SM clock como proxy |

> TODO: agregar el cuarto error del Lab 2 (ver `visual/lab2-k3s-gpu.html`).

## Patrones
- **Los mensajes de error despistan:** "did you configure the Container Toolkit?" cuando el toolkit
  estaba bien (#5).
- **Verificar cómo se instaló algo antes de configurarlo** (#7).
- **Ir a la fuente antes que a la UI** (#9).
