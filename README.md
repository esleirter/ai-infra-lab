# ai-infra-lab

Roadmap práctico hacia **AI Infrastructure Engineer** con foco en orquestación de GPU y serving de LLMs.

**Objetivo:** maximizar requests de inferencia concurrentes sobre el mínimo de GPUs al menor costo posible.

**Perfil:** Infrastructure/Platform Engineer · CNCF Kubestronaut · AWS SysOps · NVIDIA NCA-AIIO

## Hallazgos destacados

- **Time-Slicing no aísla VRAM:** Kubernetes admitió 2 pods de 2.2 GB en una GPU de 4 GB y el segundo murió por OOM → [Lab 2](labs/lab-02-k3s-gpu/)
- **Los topes de MPS se calculan sobre 4096 MiB nominales, pero CUDA ve 3.63 GiB:** dos pods dentro de su tope pueden fallar igual → [Lab 4](labs/lab-04-helm-mps/)
- **Sin métricas de profiling en GeForce, la potencia es el proxy honesto:** 20 W → 41 W bajo carga real → [Lab 3](labs/lab-03-dcgm/)
- **GPU-Util y VRAM son ejes independientes:** 2.50 GB ocupados con 0 % de util → [Lab 3](labs/lab-03-dcgm/)

---

## Estado del roadmap

Plan completo, orden de labs y certificaciones: **[ROADMAP.md](ROADMAP.md)**

| Fase | Contenido | Estado |
|---|---|---|
| **Fase 0** — Fundamentos de hardware | VRAM, Tensor Cores, HBM, cuantización | ✅ Completa |
| **Certificación** | NVIDIA NCA-AIIO | ✅ Aprobada |
| **Fase 1** — Orquestación y monitoreo | Device plugin, Time-Slicing, MPS, DCGM (home lab) · MIG solo teoría (requiere A100/H100) · Karpenter pasa a la Fase 3 | ✅ Completa (home lab) |
| **Fase 2** — Inferencia cloud native (home lab) | vLLM, observabilidad de inferencia, KServe, AI gateway, Argo Workflows + MinIO | ⏳ Siguiente |
| **Fase 3** — EKS | Terraform + Karpenter con GPU, benchmark en A10G, tensor parallelism, disaggregated serving (teoría) | ⏳ Pendiente |
| **En paralelo** | Go · AWS DevOps Engineer – Professional → Solutions Architect – Professional | ⏳ Pendiente |

## Labs

| Lab | Tema | Estado |
|---|---|---|
| [Lab 0](labs/lab-00-host-setup/) | Host: kernel, driver, DKMS, Container Toolkit | ✅ |
| [Lab 1](labs/lab-01-docker-gpu/) | Docker + GPU, validación de VRAM con TinyLlama | ✅ |
| [Lab 2](labs/lab-02-k3s-gpu/) | k3s + device plugin + Time-Slicing | ✅ |
| [Lab 3](labs/lab-03-dcgm/) | DCGM + Prometheus + Grafana | ✅ |
| [Lab 4](labs/lab-04-helm-mps/) | Device plugin por Helm + GFD/NFD + MPS | ✅ |
| Lab 5 | vLLM local + benchmark (TTFT, TPOT, concurrencia vs KV cache) | ⏳ Siguiente |
| Lab 6 | vLLM en Kubernetes + observabilidad de inferencia | ⏳ |
| Lab 7 | KServe | ⏳ |
| Lab 8 | AI gateway (Gateway API Inference Extension) | ⏳ |
| Lab 9 | Argo Workflows + MinIO | ⏳ |

## Teoría

| # | Tema |
|---|---|
| [01](teoria/01-arquitectura-gpu.md) | Arquitectura de GPU: SMs, CUDA/Tensor Cores, jerarquía de memoria, HBM, PUE |
| [02](teoria/02-memoria-y-vram.md) | Memoria de un modelo e inferencia: pesos, KV cache, GQA, PagedAttention, batching |
| [03](teoria/03-container-runtime.md) | Container runtime de NVIDIA: shim, OCI hooks, RuntimeClass |
| [04](teoria/04-kubernetes-gpu.md) | Kubernetes y GPU: device plugin, DRA, GPU Operator, Bottlerocket, NFD/GFD |
| [05](teoria/05-gpu-sharing.md) | Compartir GPU: Time-Slicing vs MPS vs MIG |
| [06](teoria/06-observabilidad.md) | Observabilidad: GPU-Util vs SM_ACTIVE, DCGM, segmento GeForce |
| [07](teoria/07-datacenter-nca-aiio.md) | Datacenter y NCA-AIIO: redes, storage, BasePOD/SuperPOD |
| [Glosario](teoria/glosario.md) | Todos los términos en 2–3 líneas |

Errores encontrados y su causa raíz: [errores.md](errores.md)

## Home lab

| Componente | Detalle |
|---|---|
| Máquina | Gigabyte B550-AORUS |
| GPU | NVIDIA GTX 1650 SUPER — TU116, Turing, SM 7.5, 4 GB GDDR6, 20 SMs, sin Tensor Cores, sin MIG |
| SO | Linux Mint 21 (base Ubuntu 22.04), kernel 6.8.0-136 |
| Driver | NVIDIA 595.84 (techo CUDA 13.2) |
| Stack | Docker 27.1.1, NVIDIA Container Toolkit 1.19.1, k3s v1.36.2+k3s1, device plugin 0.17.1 (Helm) + GFD/NFD |
| Observabilidad | dcgm-exporter, kube-prometheus-stack, Grafana |

> Las limitaciones de esta GPU (4 GB, sin Tensor Cores, sin MIG, sin métricas `DCGM_FI_PROF_*`)
> son parte del aprendizaje: obligan a entender qué mide cada cosa y qué no.

## Stack de producción objetivo

EKS · instancias GPU (g5/g6/p4d/p5) · Karpenter · FSx for Lustre · EFA · MinIO · DCGM · KServe · vLLM

## Próximos pasos

1. **Preparar la GPU para vLLM:** pasar de MPS `replicas: 2` a Time-Slicing o MPS `replicas: 1`
2. **Lab 5 — vLLM local** con `--dtype=half --max-model-len=1024 --gpu-memory-utilization=0.85 --enforce-eager`, midiendo TTFT, TPOT y throughput cambiando una variable a la vez
3. Labs 6–9 de la Fase 2, luego Fase 3 en EKS (detalle en [ROADMAP.md](ROADMAP.md))
4. En paralelo: Go y preparación de AWS DevOps Engineer – Professional
