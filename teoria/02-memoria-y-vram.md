# 02 — Memoria de un modelo e inferencia

**Fuentes:** sesiones del 1 y 4 de agosto de 2026 · Ejercicios: `visual/ejercicios-vram.html`

## Peso del modelo
**VRAM de pesos = parámetros × bytes por parámetro**

| Precisión | Bytes/param | TinyLlama 1.1B | Llama 70B |
|---|---|---|---|
| FP32 | 4 | ~4.4 GB | 280 GB |
| FP16 / BF16 | 2 | ~2.2 GB | 140 GB |
| FP8 / INT8 | 1 | ~1.1 GB | 70 GB |

Comprobado en [Lab 1](../labs/lab-01-docker-gpu/): TinyLlama FP16 = **2.20 GB** medidos; FP32 → OOM en 4 GB.

## Cuantización
Bajar la precisión de los pesos (FP16 → FP8/INT8/INT4). Dos efectos:
1. Menos VRAM → caben modelos más grandes o más KV cache.
2. En GPUs con Tensor Cores, activa pipelines de hardware más rápidos (ver [01](01-arquitectura-gpu.md)).

## Overhead real
- **Contexto CUDA:** ~54 MB medidos en la 1650 SUPER (se estimaba 400 MB).
- **Caché del allocator de PyTorch:** `memory_reserved` − `memory_allocated` (≈100 MB en el lab).

## Presupuesto de VRAM
**Pesos + KV cache + activaciones + overhead (contexto CUDA, allocator)**

## KV cache
Al generar, el modelo guarda las claves (K) y valores (V) de atención de los tokens ya procesados para
no recalcularlos.

- Crece con la **longitud de contexto** y con el **número de requests concurrentes**.
- En la práctica es lo que limita cuántas requests caben en una GPU.

### GQA vs MHA
- **MHA (Multi-Head Attention):** cada cabeza de atención tiene su propio K/V.
- **GQA (Grouped-Query Attention):** varias cabezas de query comparten K/V → KV cache mucho menor.
- Se calcula con los campos del `config.json` del modelo (capas, cabezas de atención, cabezas K/V,
  dimensión). Ejercicio con Llama-3-8B en `visual/ejercicios-vram.html`.

> TODO: agregar la fórmula exacta del KV cache usada en los ejercicios.

## Decode memory-bound
En decode (token por token) se leen todos los pesos desde memoria por cada token. Los SMs esperan
datos: el límite es ancho de banda, no cómputo.

## PagedAttention
Ataca el desperdicio de **capacidad**: gestiona el KV cache en páginas (como memoria virtual) y evita
la fragmentación. Es la base de vLLM.

## Continuous batching
Ataca el **ancho de banda**: una misma lectura de pesos sirve a N requests simultáneas. El throughput
total sube mucho mientras la latencia por token individual casi no cambia.

## Por estudiar (Fase 2)
> TODO: prefill vs decode, TTFT, TPOT/ITL, compute-bound vs memory-bound medido con vLLM (Lab 4).

## Conexión con el objetivo
Más requests concurrentes por GPU = más KV cache disponible y mejor reutilización de cada lectura de
pesos. Cuantización, PagedAttention y batching atacan exactamente eso.
