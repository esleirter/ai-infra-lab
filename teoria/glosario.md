# Glosario

Definiciones cortas. El detalle está en el archivo enlazado.

| Término | Definición | Ver |
|---|---|---|
| **SM (Streaming Multiprocessor)** | Unidad real de ejecución de la GPU; contiene CUDA Cores, Tensor Cores y L1 propios. H100: 132, 1650 SUPER: 20 | [01](01-arquitectura-gpu.md) |
| **CUDA Core** | ALU simple que opera un número a la vez; ejecuta lo "accesorio" de un LLM (activaciones, normas, softmax) | [01](01-arquitectura-gpu.md) |
| **Tensor Core** | Unidad que multiplica matrices completas por instrucción; requiere FP16/BF16/FP8 | [01](01-arquitectura-gpu.md) |
| **Warp / SIMT** | Grupo de 32 hilos que ejecutan la misma instrucción sobre datos distintos | [01](01-arquitectura-gpu.md) |
| **Warp divergence** | Un `if/else` que divide el warp obliga a ejecutar ambas ramas en serie | [01](01-arquitectura-gpu.md) |
| **Shared Memory / L1** | Memoria rápida privada de cada SM (~30 ciclos); la base de FlashAttention | [01](01-arquitectura-gpu.md) |
| **HBM** | DRAM apilada junto al die con bus muy ancho (5,120 bits, 3.35 TB/s en H100) | [01](01-arquitectura-gpu.md) |
| **GDDR** | VRAM sin apilamiento, más barata y con menos ancho de banda (A10G, RTX, 1650 SUPER) | [01](01-arquitectura-gpu.md) |
| **VRAM** | Nombre genérico de la memoria de la tarjeta gráfica | [01](01-arquitectura-gpu.md) |
| **Memory-bound** | Carga limitada por ancho de banda de memoria, no por cómputo; caso típico del decode | [01](01-arquitectura-gpu.md) |
| **FlashAttention** | Calcula la atención por bloques en Shared Memory sin escribir la matriz completa en HBM | [01](01-arquitectura-gpu.md) |
| **PUE** | Energía total del datacenter ÷ energía de TI; multiplica el costo energético | [01](01-arquitectura-gpu.md) |
| **Cuantización** | Bajar la precisión de los pesos (FP16 → FP8/INT8) para ahorrar VRAM y usar Tensor Cores | [02](02-memoria-y-vram.md) |
| **KV cache** | Claves y valores de atención guardados para no recalcularlos; crece con contexto y concurrencia | [02](02-memoria-y-vram.md) |
| **MHA / GQA** | Multi-Head vs Grouped-Query Attention; GQA comparte K/V entre cabezas y reduce el KV cache | [02](02-memoria-y-vram.md) |
| **Decode** | Fase de generación token por token; lee todos los pesos por token | [02](02-memoria-y-vram.md) |
| **PagedAttention** | Gestión del KV cache en páginas para evitar fragmentación (vLLM) | [02](02-memoria-y-vram.md) |
| **Continuous batching** | Reutiliza una lectura de pesos para N requests simultáneas | [02](02-memoria-y-vram.md) |
| **Contexto CUDA** | Estado que crea el driver por proceso; ~54 MB medidos | [02](02-memoria-y-vram.md) |
| **nvidia-container-runtime** | Shim entre containerd y runc que inyecta GPU en el OCI spec | [03](03-container-runtime.md) |
| **libnvidia-container** | Librería que realiza la inyección de dispositivos y librerías | [03](03-container-runtime.md) |
| **Prestart hook (OCI)** | Gancho que se ejecuta antes de arrancar el contenedor; ahí se inyecta la GPU | [03](03-container-runtime.md) |
| **nvidia-ctk** | CLI del Container Toolkit para registrar el runtime en Docker/containerd | [03](03-container-runtime.md) |
| **RuntimeClass** | Objeto de Kubernetes que elige el runtime de un pod | [03](03-container-runtime.md) |
| **NVIDIA_VISIBLE_DEVICES** | Variable que el device plugin inyecta para indicar qué GPU usar | [03](03-container-runtime.md) |
| **Device plugin** | Componente que anuncia `nvidia.com/gpu` al kubelet | [04](04-kubernetes-gpu.md) |
| **DRA** | Dynamic Resource Allocation: asignación por ResourceClaims; k8s 1.34+ | [04](04-kubernetes-gpu.md) |
| **GPU Operator** | Instala y gestiona todo el stack GPU en Kubernetes; no recomendado en EKS | [04](04-kubernetes-gpu.md) |
| **Bottlerocket** | SO inmutable de AWS para contenedores con updates A/B | [04](04-kubernetes-gpu.md) |
| **NFD / GFD** | Node/GPU Feature Discovery: etiquetan nodos con características del hardware | [04](04-kubernetes-gpu.md) |
| **EKS Hybrid Nodes** | Nodos on-prem unidos a un control plane de EKS | [04](04-kubernetes-gpu.md) |
| **Time-Slicing** | Compartición temporal de la GPU sin aislamiento de memoria | [05](05-gpu-sharing.md) |
| **MPS** | Un daemon con un único contexto CUDA para varios procesos; SMs compartidos en espacio | [05](05-gpu-sharing.md) |
| **MIG** | Partición física de SMs, L2 y memoria; solo datacenter | [05](05-gpu-sharing.md) |
| **GPU-Util** | % del tiempo con algún kernel activo; no es ocupación de SMs | [06](06-observabilidad.md) |
| **DCGM** | Data Center GPU Manager: gestión y métricas de GPU | [06](06-observabilidad.md) |
| **`DCGM_FI_PROF_SM_ACTIVE`** | Fracción real de SMs con trabajo; solo datacenter | [06](06-observabilidad.md) |
| **dcgm-exporter** | Expone métricas DCGM a Prometheus con atribución por pod | [06](06-observabilidad.md) |
| **InfiniBand / RoCE** | Redes de baja latencia con RDMA para clusters de IA | [07](07-datacenter-nca-aiio.md) |
| **GPUDirect RDMA** | Transferencia directa entre GPU y red sin pasar por la CPU | [07](07-datacenter-nca-aiio.md) |
| **DGX BasePOD / SuperPOD** | Arquitecturas de referencia de NVIDIA para clusters de IA | [07](07-datacenter-nca-aiio.md) |
| **SLURM** | Scheduler de trabajos HPC, alternativa a Kubernetes para entrenamiento | [07](07-datacenter-nca-aiio.md) |
