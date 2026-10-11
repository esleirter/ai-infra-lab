# Lab 1 — Docker + GPU y validación de VRAM

**Fecha:** 4 de agosto de 2026 · **Teoría relacionada:** [02 — Memoria y VRAM](../../teoria/02-memoria-y-vram.md), [03 — Container runtime](../../teoria/03-container-runtime.md)

## Objetivo
Correr cargas GPU en contenedores y comprobar que el cálculo teórico de VRAM coincide con la realidad.

## Pasos

### 1. Prueba básica
```bash
docker run --rm --gpus all ubuntu:22.04 nvidia-smi
```
La imagen no trae drivers: el runtime de NVIDIA los inyecta desde el host.

### 2. Aislamiento de PID namespace
Desde dentro del contenedor `nvidia-smi` no muestra los procesos del host, pero **sí** la VRAM
total usada. La memoria de la GPU es un recurso compartido que el namespace no oculta.

### 3. TinyLlama-1.1B-Chat en FP16
Imagen: `pytorch/pytorch:2.6.0-cuda12.4-cudnn9-runtime`

Predicción: 1.1B parámetros × 2 bytes (FP16) ≈ **2.2 GB**.

| Medición | Valor |
|---|---|
| `torch.cuda.memory_allocated()` | **2.20 GB** |
| `torch.cuda.memory_reserved()` | 2.30 GB (≈100 MB de caché del allocator) |
| Overhead contexto CUDA | **~54 MB** (se estimaba 400 MB) |

### 4. TinyLlama en FP32
1.1B × 4 bytes ≈ 4.4 GB > 4 GB disponibles → **OOM**, como se esperaba.

El modelo se cargó de forma interactiva dentro del contenedor, con el caché de Hugging Face montado:
```bash
docker run --rm --gpus all -it \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  pytorch/pytorch:2.6.0-cuda12.4-cudnn9-runtime bash
```
> TODO: el script Python no se guardó (se escribió dentro del contenedor); reescribirlo en `scripts/`
> cuando se repita la medición.

## GPU-Util durante la generación
- Marcaba ~98%, no 100%: el overhead de Python entre llamadas a `generate()` deja huecos sin kernel.
- GPU-Util = porcentaje del tiempo de muestreo con *algún* kernel activo. **No** es ocupación de SMs
  (eso es `DCGM_FI_PROF_SM_ACTIVE`, ver [06](../../teoria/06-observabilidad.md)).

## Entregable
`visual/ejercicios-vram.html`: 13 ejercicios en 4 niveles (peso del modelo, KV cache con GQA vs MHA
usando el `config.json` de Llama-3-8B, presupuesto total de VRAM).

## Lecciones
1. La fórmula parámetros × bytes por parámetro funciona con precisión.
2. El overhead del contexto CUDA es mucho menor de lo que se suele estimar.
3. VRAM se reparte entre pesos, KV cache, activaciones y caché del allocator.
