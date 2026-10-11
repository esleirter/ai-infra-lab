# Roadmap — ai-infra-lab

**Última actualización:** 11 de octubre de 2026

**Objetivo:** maximizar requests de inferencia concurrentes sobre el mínimo de GPUs al menor costo posible.
**Meta profesional:** roles remotos en EE. UU. de AI/ML Infrastructure Engineer (serving).

> Regla de trabajo: construir, medir, cambiar una variable a la vez y documentar el cuello de botella,
> el tradeoff y el resultado. Cada lab termina con un write-up con números.

---

## 📍 Dónde estamos

- **Fase 1 cerrada en el home lab** (Labs 0–4). Último lab: device plugin por Helm + GFD/NFD + MPS.
- **Siguiente: Lab 5 — vLLM local**, inicio de la Fase 2.
- **Antes del Lab 5:** el cluster está en MPS `replicas: 2` (tope ~2 GB por pod). Hay que volver a
  Time-Slicing o dejar MPS `replicas: 1` para que vLLM tenga la GPU completa.

---

## Fase 0 — Fundamentos de hardware ✅
VRAM, SMs, CUDA/Tensor Cores, HBM, cuantización, memory-bound. → [teoria/01](teoria/01-arquitectura-gpu.md), [teoria/02](teoria/02-memoria-y-vram.md)

## Certificación NVIDIA NCA-AIIO ✅

## Fase 1 — Orquestación y monitoreo de GPU ✅ (home lab)

| Lab | Tema | Estado |
|---|---|---|
| [0](labs/lab-00-host-setup/) | Host: kernel, driver, DKMS, Container Toolkit | ✅ |
| [1](labs/lab-01-docker-gpu/) | Docker + GPU, validación de VRAM | ✅ |
| [2](labs/lab-02-k3s-gpu/) | k3s + device plugin + Time-Slicing | ✅ |
| [3](labs/lab-03-dcgm/) | DCGM + Prometheus + Grafana | ✅ |
| [4](labs/lab-04-helm-mps/) | Device plugin por Helm + GFD/NFD + MPS | ✅ |

Fuera del alcance del home lab:
- **MIG:** solo teoría (requiere A100/H100).
- **Karpenter:** pasa a la Fase 3 (requiere EKS).
- Opcional: prueba de cómputo con MPS (`active thread` 50 % con `burn`).

---

## Fase 2 — Inferencia cloud native (home lab) ⏳

| Lab | Tema | Qué se aprende |
|---|---|---|
| **5** | vLLM local + benchmark | TTFT, TPOT, throughput y concurrencia vs KV cache, cambiando una variable a la vez. Teoría previa: prefill vs decode, crecimiento del KV cache. |
| **6** | vLLM en Kubernetes + observabilidad de inferencia | De `docker run` a Deployment. Métricas Prometheus de vLLM (TTFT, uso de KV cache, cola de requests) en Grafana: monitorear el servicio, no solo la GPU. |
| **7** | KServe | Abstracción de serving: escalado a cero, canary entre versiones de modelo, autoscaling por concurrencia. |
| **8** | AI gateway | Routing, rate limits y fallbacks delante de los modelos con la Gateway API Inference Extension. |
| **9** | Argo Workflows + MinIO | Pipeline que descarga el modelo, lo guarda en MinIO y el serving lo carga desde ahí. |

- **KubeRay:** teoría o prueba corta (con 4 GB de VRAM hay poco que distribuir).

---

## Fase 3 — EKS (lo que el home lab no puede) ⏳

1. **Terraform: EKS + Karpenter con NodePools de GPU** (g5/g6), escalando nodos según la demanda.
   Cubre el gap de IaC aplicado a GPU.
2. **Mismo benchmark del Lab 5 en una A10G (24 GB):** comparación contra la GTX 1650 SUPER en
   KV cache, concurrencia y costo por token.
3. **Paralelismo:** modelo que no cabe en una GPU, servido con tensor parallelism en una g5.12xlarge (4 GPUs).
4. **Disaggregated serving** (prefill/decode separados, llm-d): principalmente teoría.

Regla de costo: sesiones cortas, instancias spot y `terraform destroy` al terminar cada sesión.

---

## En paralelo

### Go
Fundamentos, apuntando a escribir algo útil para el lab (un exporter o un controlador pequeño).

### Certificaciones AWS

| Orden | Certificación | Por qué |
|---|---|---|
| 1 | **DevOps Engineer – Professional** | Continuación natural de SysOps (IaC, CI/CD, monitoreo, automatización); se prepara junto con la Fase 3. Renueva SysOps Associate. |
| 2 | **Solutions Architect – Professional** | Diseño de redes de baja latencia, almacenamiento de alto rendimiento (FSx for Lustre) y costo a escala. Al cerrar la Fase 3. |
| Opcional | Machine Learning Engineer – Associate | Foco en SageMaker/MLOps más que en GPUs en Kubernetes. Verificar versión vigente (MLA-C02 reemplaza a MLA-C01 desde fines de septiembre de 2026). |
| No por ahora | Generative AI Developer – Professional | Orientada a desarrollo de aplicaciones con Bedrock, no a operar infraestructura de inferencia. |

Nota: la Machine Learning – Specialty fue retirada el 31 de marzo de 2026.

Las certificaciones sirven para pasar filtros; lo que diferencia en entrevistas es el portafolio con
números. Certificar en paralelo con lo que se construye, sin frenar los labs.

---

## Lecturas
1. *GPU-Enabled Platforms on Kubernetes* (vcluster.com) — consolidar Fase 1.
2. *Right-Sizing GPUs in Kubernetes* (learnkube.com) — después de vLLM con métricas reales.

(Ambos patrocinados: núcleo técnico sólido, sesgo en capítulos finales.)
