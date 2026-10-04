# 01 — Arquitectura de GPU

**Fuente:** sesión del 1 de agosto de 2026 (NCA-AIIO) · Referencia principal: H100 SXM

## CPU vs GPU
La CPU tiene pocos núcleos complejos optimizados para trabajo secuencial (predicción de saltos,
ejecución fuera de orden). La GPU tiene miles de núcleos simples que aplican la misma operación sobre
muchos datos a la vez: exactamente lo que piden las multiplicaciones de matrices de un LLM.

## SM — Streaming Multiprocessor
La **unidad real de ejecución** de la GPU. No es "un procesador con miles de núcleos": es un conjunto de
mini-procesadores independientes que reciben bloques de trabajo y los ejecutan en paralelo sin
coordinarse entre sí.

- H100: 132 SMs · GTX 1650 SUPER: 20 SMs.
- **Analogía:** la GPU es el clúster, cada SM es un nodo, y el scheduler de hardware (GigaThread
  Engine) reparte *thread blocks* entre SMs como el kube-scheduler reparte pods.

### Dentro de cada SM (H100)
| Componente | Función |
|---|---|
| 128 CUDA Cores FP32 | Aritmética escalar elemento a elemento |
| 4 Tensor Cores (4ª gen) | Multiplicación de matrices completas por instrucción (FP16/BF16/FP8) |
| 256 KB L1 / Shared Memory | Caché privada del SM, configurable |
| 4 Warp Schedulers | Gestionan *warps* de 32 hilos |

Total H100: 132 × 128 = 16,896 CUDA Cores y 528 Tensor Cores.

## CUDA Cores
ALU simple: **una operación sobre un número a la vez**. Mucho más simple que un core de CPU, pero hay
miles.

- Modo **SIMT**: los 32 hilos de un warp ejecutan la misma instrucción sobre datos distintos.
- **Warp divergence:** si un `if/else` divide el warp, se ejecutan ambas ramas en serie.
- En un LLM hacen lo "accesorio": activaciones (SiLU, GELU), LayerNorm/RMSNorm, softmax, sumas residuales.

## Tensor Cores
Operan sobre **matrices completas en una instrucción**: `D = A×B + C` en bloques.

| Unidad (H100 SXM) | Rendimiento |
|---|---|
| CUDA Cores FP32 | ~67 TFLOPS |
| Tensor Cores FP8 | ~3,958 TFLOPS (con sparsity) |

- La cuantización no solo reduce VRAM: **desbloquea un pipeline de hardware más rápido**.
- Un modelo en FP32 puro no toca los Tensor Cores.
- Aun así, en inferencia de LLM los Tensor Cores casi nunca son el límite (ver memory-bound abajo).
- La GTX 1650 SUPER (TU116) **no tiene Tensor Cores**.

## Jerarquía de memoria
| Nivel | Latencia aprox. |
|---|---|
| Registros | ~1 ciclo |
| L1 / Shared Memory (por SM) | ~30 ciclos |
| L2 (compartida, 50 MB en H100) | ~200 ciclos |
| HBM3 (80 GB en H100) | ~500+ ciclos |

- En H100, L1 y Shared Memory son un bloque configurable: parte es caché automática y parte memoria
  que el kernel CUDA controla explícitamente.
- **FlashAttention** usa ese control: procesa la atención por bloques que caben en Shared Memory en vez
  de escribir la matriz completa en HBM. No cambia las matemáticas, cambia dónde vive el dato.

## DRAM, HBM y VRAM
- **DRAM:** tecnología base. Guarda cada bit como carga en un condensador que se fuga y hay que
  refrescar. Más lenta que la SRAM (de la que están hechas L1/L2), pero barata y densa.
- **HBM (High Bandwidth Memory):** DRAM apilada en torres de 8–12 chips con micro-vías (TSVs), en el
  mismo sustrato que la GPU. El truco es la **anchura del bus**, no la velocidad de cada chip.
- **VRAM:** nombre genérico de la memoria de la tarjeta. En H100 es HBM3; en A10G (g5), RTX 4090 o la
  1650 SUPER es GDDR.

| Memoria | Bus | Ancho de banda |
|---|---|---|
| DDR5 (servidor) | 64 bits/canal | ~50–100 GB/s |
| GDDR6X (RTX 4090) | 384 bits | ~1 TB/s |
| HBM3 (H100 80 GB) | 5,120 bits | 3.35 TB/s |
| HBM3e (H200 141 GB) | 6,144 bits | 4.8 TB/s |

## Capacidad vs ancho de banda
Dos límites distintos:

- **Capacidad (GB) → ¿entra el modelo?** Pesos + KV cache + activaciones. Llama 70B en FP16 = 140 GB:
  no cabe en una H100 de 80 GB. En FP8 = 70 GB: cabe, con poco margen para KV cache.
- **Ancho de banda (TB/s) → ¿qué tan rápido genera?** En decode se leen **todos los pesos por cada
  token**. 70 GB a 3.35 TB/s ≈ techo de ~47 tokens/s por secuencia. Ninguna optimización de cómputo lo
  cambia.

**Decode es memory-bound:** los SMs pasan la mayor parte del tiempo esperando pesos desde HBM.
Optimizar inferencia es, en la mayoría de los casos, un problema de mover datos, no de FLOPS.
Ver [02 — Memoria y VRAM](02-memoria-y-vram.md).

## Selección de instancias
- p5 (H100, HBM3): modelos grandes con alta concurrencia.
- g5/g6 (GDDR6): modelos de 7–13B donde el ancho de banda no ahoga.

## PUE — Power Usage Effectiveness
Energía total del datacenter ÷ energía de los equipos de TI. Multiplica directamente el costo energético
de las GPUs.

- Direct Liquid Cooling: refrigerante en contacto con los chips → PUE < 1.2.
- Racks de 60–120 kW (como GB300 NVL72) ya usan refrigeración líquida completa.

## Conexión con el objetivo
Saber si una carga es compute-bound o memory-bound decide qué optimizar (cuantización, batching) y qué
instancia pagar. Medido en [Lab 3](../labs/lab-03-dcgm/): GPU-Util no distingue entre ambas.
