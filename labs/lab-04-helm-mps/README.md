# Lab 4 — Device plugin a Helm + GFD/NFD + MPS

**Fecha:** 11 de octubre de 2026 · **Teoría relacionada:** [04 — Kubernetes y GPU](../../teoria/04-kubernetes-gpu.md), [05 — Compartir GPU](../../teoria/05-gpu-sharing.md)

## Objetivo
Migrar el device plugin de manifiesto crudo a Helm (sin cambiar nada más), obtener las labels de GFD y
pasar de Time-Slicing a MPS para medir qué aísla y qué no.

---

## Parte A — Migración a Helm (una variable a la vez)
Se mantiene el mismo Time-Slicing (`replicas: 4`) para que lo único que cambie sea el método de instalación.

**Motivo:** MPS necesita el DaemonSet `mps-control-daemon` y la label `nvidia.com/mps.capable=true`,
que la pone GFD. El manifiesto crudo no trae ninguno de los dos.

### 1. Backup y borrado del manifiesto crudo
DaemonSet y ConfigMap del Lab 2 guardados en
[`../lab-02-k3s-gpu/manifests/backup/`](../lab-02-k3s-gpu/manifests/backup/); después, delete del
manifiesto v0.17.1 y del ConfigMap de Time-Slicing.

### 2. Instalar por Helm
[`values-device-plugin.yaml`](values-device-plugin.yaml): `runtimeClassName: nvidia`, `gfd.enabled: true`
(instala GFD y NFD como subchart) y `config.map.default` con `timeSlicing` `replicas: 4`.
```bash
helm install nvdp nvdp/nvidia-device-plugin --version 0.17.1 \
  -n nvidia-device-plugin --create-namespace \
  -f values-device-plugin.yaml
```
Pods resultantes: NFD master + worker, GFD, device plugin (y luego `mps-control-daemon`).

### 3. Resultado
Allocatable `nvidia.com/gpu: 4`, igual que antes. Lo nuevo son las labels de GFD:

| Label (`nvidia.com/`) | Valor | Nota |
|---|---|---|
| `gpu.count` | 1 | GPU física; Allocatable 4 son réplicas |
| `gpu.product` | `NVIDIA-GeForce-GTX-1650-SUPER-SHARED` | Sufijo `-SHARED` por el sharing |
| `gpu.sharing-strategy` | `time-slicing` | |
| `gpu.replicas` | 4 | |
| `gpu.compute.major` / `minor` | 7 / 5 | |
| `gpu.family` | `turing` | |
| `gpu.memory` | 4096 | MiB nominales |
| `gpu.mode` | `graphics` | |
| `mig.capable` | `false` | |
| `mps.capable` | `false` | Pasa a `true` en la Parte B |

GFD consulta la GPU vía NVML y etiqueta el nodo; corre sobre NFD, el framework general de labels de
hardware/kernel. Antes solo existían las labels manuales de k3s (`gpu=true`, `gpu.model=gtx1650super`).

---

## Parte B — MPS
[`values-device-plugin-mps.yaml`](values-device-plugin-mps.yaml): `sharing.mps` con `replicas: 2`.
```bash
helm upgrade nvdp nvdp/nvidia-device-plugin --version 0.17.1 \
  -n nvidia-device-plugin -f values-device-plugin-mps.yaml
```

### Verificación
- Pod `mps-control-daemon` Running.
- Allocatable `nvidia.com/gpu: 2`; labels `gpu.sharing-strategy=mps`, `mps.capable=true`.
- `nvidia-smi`: `Compute M.` = `E. Process` (Exclusive Process). El único proceso tipo C es
  `nvidia-cuda-mps-server` (28 MiB); los pods aparecen como `M+C`.

**Efecto secundario:** con MPS activo, CUDA fuera de Kubernetes (`docker run --gpus all`) probablemente
falle por el modo exclusivo. Xorg y DCGM no se afectan.

### Pod de prueba
[`hog.yaml`](hog.yaml): PyTorch 2.6.0, reserva `GB` gigabytes con `torch.empty` y duerme. Se parametriza
con la variable de entorno `GB`.

### Experimento 1 — `hog-1` solo, 2.2 GB, `replicas: 2`
OOM: `total capacity of 3.63 GiB of which 1.96 GiB is free`. Con Time-Slicing el mismo pod funcionaba.

| Cálculo | MiB |
|---|---|
| Tope por réplica = 4096 / 2 | 2048 |
| Contexto del cliente | −42 |
| Disponible para el pod | 2006 ≈ 1.96 GiB |

El tope es fijo **por réplica configurada**, no por pods corriendo: la memoria de la réplica vacía no se
presta. Más aislamiento = menos aprovechamiento.

### Experimento 2 — `hog-1` y `hog-2` con 1.5 GB, `hog-3` con 0.5 GB
- `hog-1` y `hog-2` reservaron 1.50 GB cada uno; `nvidia-smi` muestra 1578 MiB por proceso
  (1536 + 42 de contexto: cada pod paga el suyo).
- `hog-3` queda `Pending`: no hay réplicas.
- Costo fijo con MPS: servidor de 28 MiB pagado una vez + ~42 MiB por cliente
  (vs ~54 MiB de contexto completo por pod con Time-Slicing).

### Experimento 3 — `hog-1` y `hog-2` con 1.9 GB
`hog-1` OK (1988 MiB). `hog-2` OOM: `1.36 GiB is free`, **dentro de su tope**.

| Ocupado | MiB |
|---|---|
| Xorg + Cinnamon | ~248 |
| MPS server | 28 |
| hog-1 | 1988 |
| Contexto de hog-2 | 42 |
| **Total** | **~2306** |
| Usable por CUDA (3.63 GiB) | ~3717 |
| Libre para hog-2 | ~1411 ≈ 1.36 GiB |

El cliente recibe el **mínimo** entre lo que le queda de su tope y la memoria física libre.

**Causa:** los topes se calculan sobre 4096 MiB nominales, pero el driver reserva ~380 MiB (CUDA ve
3.63 GiB). Suma de topes 4096 > ~3717 usable: aun sin escritorio no cabrían dos pods llenos. El escritorio
lo agrava.

### El fallo de `hog-2`, en los dos labs
| | Time-Slicing (Lab 2) | MPS (Exp. 3) |
|---|---|---|
| Qué pasó | Sin tope, el vecino tomó todo | Ambos dentro del tope, pero la suma excede la memoria real |
| ¿Se puede evitar? | No | Sí, con réplicas bien calculadas |

## Errores y notas
- Los pods `hog` quedan en `Terminating` ~30 s: `python` es PID 1 y no maneja SIGTERM. Para labs,
  `terminationGracePeriodSeconds: 0` en `hog.yaml`.

## Estado final del cluster
Device plugin por Helm (release `nvdp`, namespace `nvidia-device-plugin`), MPS `replicas: 2`, GFD/NFD activos.

## Lecciones
1. El tope de memoria de MPS es un **límite máximo, no una reserva garantizada**. La garantía existe
   solo si la suma de topes cabe en la memoria usable: calcular réplicas sobre memoria usable, no nominal.
2. El tope es por réplica configurada: lo que una réplica vacía no usa, no se presta.
3. MPS cobra un contexto compartido (28 MiB) más ~42 MiB por cliente; Time-Slicing cobra ~54 MiB por pod.
4. GFD convierte el hardware en labels (`gpu.count` dice 1 aunque Allocatable diga 4).

## Pendiente
- Prueba de cómputo con MPS (`active thread` 50 % con `burn`), opcional.

## Siguiente
Lab 5 — vLLM.
