# 03 — Container runtime de NVIDIA

**Fuentes:** sesiones del 27 de junio y 4 de agosto de 2026

## Driver vs CUDA toolkit
- El **driver** vive en el **host**. Define el techo de versión de CUDA (595.84 → CUDA 13.2).
- El **CUDA toolkit / runtime** vive **dentro de la imagen** del contenedor.
- Por eso una imagen `ubuntu:22.04` sin nada de NVIDIA puede correr `nvidia-smi`: el runtime le inyecta
  las librerías del driver del host.

## nvidia-container-runtime
Un **shim** entre el runtime de alto nivel (containerd, Docker) y el de bajo nivel (`runc`):

```
containerd → nvidia-container-runtime → (modifica el OCI spec) → runc
```

- Usa `libnvidia-container` y un **prestart hook** para agregar al OCI spec los dispositivos
  `/dev/nvidia*` y las librerías del driver.
- Después delega en `runc` real.
- Por eso en k3s aparece `runtime_type = "io.containerd.runc.v2"` con
  `BinaryName = "/usr/bin/nvidia-container-runtime"`: al final es runc, solo cambia quién prepara el spec.

## nvidia-ctk
Herramienta del Container Toolkit que registra el runtime:
```bash
sudo nvidia-ctk runtime configure --runtime=docker      # o containerd
```
Requiere que `/etc/docker/daemon.json` sea JSON válido (ver [Lab 0](../labs/lab-00-host-setup/)).

## RuntimeClass (Kubernetes)
Indica con qué runtime ejecutar un pod. En un pod GPU hacen falta dos piezas:

| Pieza | Función |
|---|---|
| `resources.limits: nvidia.com/gpu: 1` | **La solicitud.** El scheduler asigna y el device plugin inyecta `NVIDIA_VISIBLE_DEVICES` |
| `runtimeClassName: nvidia` | **Quién ejecuta.** El runtime lee esa variable e inyecta el dispositivo |

Si `nvidia` es el runtime por defecto de containerd, basta con la solicitud (es lo que hacen el GPU
Operator y las AMIs GPU de EKS). Comprobado en [Lab 2](../labs/lab-02-k3s-gpu/).

## Casos de plataforma
- **Mac (Apple Silicon):** el bloqueo no es ARM, es que no hay GPU NVIDIA. El Mac funciona como plano de
  control (`kubectl`, SSH, Terraform) y el cómputo vive en nodos cloud (g4dn/g5/g6).
- **WSL2:** el driver vive en Windows y proyecta `libcuda.so` hacia Linux. Docker y el Container Toolkit
  se instalan dentro de WSL2. **Nunca instalar un driver NVIDIA de Linux dentro de WSL2.**
- **EKS:** no se instala el toolkit nodo por nodo; lo traen las AMIs optimizadas para GPU (o el GPU
  Operator fuera de EKS).

## Conexión con el objetivo
Entender la cadena host → contenedor → Kubernetes es lo que permite diagnosticar en minutos por qué un
pod "tiene GPU asignada" pero no la ve.
