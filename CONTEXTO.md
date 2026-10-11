# CONTEXTO — ai-infra-lab

> Resumen condensado para dar contexto a Claude (Project "IA Infraestructure" y Claude Code).
> Actualizar después de cada lab. Detalle completo en `labs/` y `teoria/`.

**Última actualización:** 11 de octubre de 2026

## Quién y para qué
- Esleiter Vilchez — Infrastructure/Platform Engineer (Perú), Kubestronaut, AWS SysOps, NVIDIA NCA-AIIO.
- Meta: roles remotos en EE. UU. de AI/ML Infrastructure Engineer (serving).
- Objetivo técnico: máximas requests de inferencia concurrentes sobre el mínimo de GPUs al menor costo.
- Gaps detectados en ofertas de EE. UU.: Terraform/IaC aplicado a GPU y Go como lenguaje de sistemas.

## Convenciones
- Respuestas técnicas en español, directas y precisas.
- Resúmenes de material de curso: solo lo que dice la fuente, sin agregar conocimiento externo.
- Visualizaciones: HTML/SVG tema oscuro (#0f0f13), colores por tipo (azul datos, teal procesos,
  púrpura resultados, coral alertas, ámbar notas, gris contexto), cierre conectando con el objetivo.

## Home lab
GTX 1650 SUPER (TU116, SM 7.5, 4 GB GDDR6, 20 SMs, sin Tensor Cores, sin MIG, GeForce → sin
`DCGM_FI_PROF_*`) · Linux Mint 21, kernel 6.8.0-136 · driver 595.84 (CUDA 13.2) · Docker 27.1.1 ·
NVIDIA Container Toolkit 1.19.1 · k3s v1.36.2+k3s1 · device plugin 0.17.1 por Helm (release `nvdp`, ns `nvidia-device-plugin`) + GFD/NFD.

## Estado
- Fase 0 ✅ · NCA-AIIO ✅ · Fase 1 ✅ (home lab; MIG solo teoría, Karpenter → Fase 3) · Fase 2 siguiente ·
  Fase 3 (EKS) pendiente. Plan completo en `ROADMAP.md`.
- Lab 0 host ✅ · Lab 1 Docker+GPU ✅ · Lab 2 k3s+Time-Slicing ✅ · Lab 3 DCGM ✅ ·
  Lab 4 Helm+GFD/NFD+MPS ✅ · Lab 5 vLLM ⏳.
- Cluster: device plugin por Helm con **MPS `replicas: 2`** (antes Time-Slicing `replicas: 4`; backup del
  manifiesto crudo en `labs/lab-02-k3s-gpu/manifests/backup/`), GFD/NFD activos, dcgm-exporter +
  kube-prometheus-stack (`monitoring`), Grafana NodePort 30300, dashboard 12239.
- Con MPS activo, CUDA fuera de k8s (`docker run --gpus all`) probablemente falle (modo `E. Process`).
- Runtime por defecto de containerd sigue siendo `runc` (no existe `config.toml.tmpl`, verificado 11 oct):
  decisión consciente, todo pod GPU declara `runtimeClassName: nvidia`.

## Hallazgos clave (medidos)
1. TinyLlama-1.1B FP16 = 2.20 GB allocated; contexto CUDA ~54 MB; FP32 → OOM.
2. Time-Slicing no aísla VRAM: k8s admitió hog-1 y hog-2, hog-2 murió por OOM (3.63 GiB totales).
3. ConfigMap montado como volumen: el key es el filename → con manifiesto crudo el key es `config.yaml`
   (Helm usa `any` porque su sidecar config-manager lee por API).
4. El método de instalación determina el de configuración.
5. Un pod GPU en k3s necesita `nvidia.com/gpu` (solicitud) **y** `runtimeClassName: nvidia` (inyección),
   salvo que `nvidia` sea el runtime por defecto. El pod del device plugin también necesita el RuntimeClass.
6. GPU-Util y VRAM son ejes independientes (2.50 GB ocupados con 0% util).
7. Sin métricas de profiling, la potencia es el proxy honesto: 20.2 W idle vs 40.7 W bajo carga;
   SM clock 1.20 → 1.92 GHz.
8. `DCGM_FI_PROF_*` es restricción de segmento (GeForce vs datacenter), no de arquitectura.
9. Ante datos que "no aparecen" en Grafana, consultar la API de Prometheus directamente.
10. MPS pone un **tope por cliente** = nominal/réplicas (2048 MiB) − ~42 MiB de contexto; es fijo por
    réplica configurada, no por pods corriendo (la réplica vacía no presta memoria).
11. El tope de MPS es **límite máximo, no garantía**: el cliente recibe el mínimo entre su tope y la
    memoria física libre. Topes sobre 4096 nominales vs ~3717 MiB usables (driver reserva ~380) → hog-2
    hizo OOM dentro de su tope. Calcular réplicas sobre memoria usable.
12. Costo fijo: MPS = 28 MiB de servidor + ~42 MiB por cliente; Time-Slicing = ~54 MiB por pod.
13. GFD etiqueta por NVML: `gpu.count=1` aunque Allocatable diga 4; `gpu.product` lleva sufijo `-SHARED`.
    MPS requiere `mps-control-daemon` y `mps.capable=true` (GFD): el manifiesto crudo no sirve, hay que ir a Helm.

## Conceptos base
- Decode es memory-bound: el límite es ancho de banda de memoria, no FLOPS.
- PagedAttention ataca capacidad (fragmentación del KV cache); continuous batching ataca ancho de banda.
- Time-Slicing = tiempo, sin aislamiento · MPS = un contexto CUDA compartido, SMs en espacio, tope de
  memoria por cliente (máximo, no garantía) ·
  MIG = partición física estática (solo datacenter). Se pueden apilar.
- Device plugin es la elección para Fase 1; DRA requiere k8s 1.34+, MIG/TS en alpha, no compatible con
  Karpenter dinámico.
- GPU Operator no recomendado en EKS (las AMIs ya traen driver y toolkit).

## Próximos pasos
1. Antes del Lab 5: pasar de MPS `replicas: 2` a Time-Slicing o MPS `replicas: 1` (vLLM necesita la GPU completa).
2. Lab 5: vLLM local (`--dtype=half --max-model-len=1024 --gpu-memory-utilization=0.85 --enforce-eager`)
   como benchmark: TTFT, TPOT, throughput y concurrencia vs KV cache, cambiando una variable a la vez.
   Teoría previa: prefill vs decode, crecimiento del KV cache.
3. Opcional: prueba de cómputo con MPS (`active thread` 50% con `burn`).

## Roadmap (detalle en `ROADMAP.md`)
- **Fase 2 (home lab):** Lab 5 vLLM local · Lab 6 vLLM en k8s + observabilidad de inferencia (métricas
  de vLLM en Grafana) · Lab 7 KServe · Lab 8 AI gateway (Gateway API Inference Extension) ·
  Lab 9 Argo Workflows + MinIO · KubeRay solo teoría o prueba corta.
- **Fase 3 (EKS):** Terraform EKS + Karpenter con NodePools GPU (g5/g6) · benchmark del Lab 5 en A10G
  (24 GB) con costo por token · tensor parallelism en g5.12xlarge · disaggregated serving (llm-d) teoría.
  Regla de costo: spot, sesiones cortas, `terraform destroy`.
- **En paralelo:** Go (exporter o controlador pequeño) · AWS DevOps Engineer – Professional (con Fase 3)
  → Solutions Architect – Professional (al cerrar Fase 3) · ML Engineer – Associate opcional ·
  GenAI Developer – Professional no por ahora (Bedrock, no infra). ML Specialty retirada (31 mar 2026).

## Lecturas
1. *GPU-Enabled Platforms on Kubernetes* (vcluster.com) — consolidar Fase 1.
2. *Right-Sizing GPUs in Kubernetes* (learnkube.com) — después de vLLM con métricas reales.
(Ambos patrocinados: núcleo técnico sólido, sesgo en capítulos finales.)
