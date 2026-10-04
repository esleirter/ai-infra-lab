# 04 — Kubernetes y GPU

**Fuentes:** sesiones del 4, 6 y 31 de agosto de 2026

## Las tres capas
| Capa | Componentes |
|---|---|
| Host | Driver NVIDIA, kernel module |
| Contenedor | NVIDIA Container Toolkit, runtime |
| Kubernetes | Device plugin (o DRA), scheduler, RuntimeClass |

## Device plugin
- Anuncia el recurso extendido `nvidia.com/gpu` al kubelet.
- El pod del plugin **necesita ver la GPU él mismo** para contarla: debe correr con el runtime de NVIDIA.
- Instalable por manifiesto crudo o por Helm, y **cada método se configura distinto** (ver [Lab 2](../labs/lab-02-k3s-gpu/)).

## DRA — Dynamic Resource Allocation
Mecanismo más nuevo basado en `ResourceClaim` / `ResourceClaimTemplate`.

| | Device plugin | DRA |
|---|---|---|
| MIG / Time-Slicing | GA | Alpha |
| Versión de Kubernetes | Cualquiera | 1.34+ |
| Karpenter dinámico | Compatible | No compatible |

**Decisión:** device plugin para la Fase 1. DRA es relevante para decisiones de arquitectura y
certificaciones AWS Professional, no para el NCA-AIIO.

## Bottlerocket
SO de AWS para contenedores: raíz inmutable, actualizaciones atómicas A/B, configuración por API.
Desde 1.63.0, `settings.kubelet-device-plugins.nvidia.enabled = false` desactiva su device plugin
integrado (necesario para usar DRA).

## GPU Operator
Instala y gestiona driver, toolkit, device plugin, DCGM, etc. como un conjunto.
**No recomendado en EKS:** las AMIs GPU ya traen driver y toolkit, y la propiedad de cada componente
cambia entre on-prem y EKS gestionado. Evita el error del RuntimeClass porque lo aplica a todos sus
componentes.

## NFD y GFD
Node Feature Discovery y GPU Feature Discovery etiquetan los nodos con características del hardware.
> TODO: completar tras instalarlos vía Helm.

## EKS Hybrid Nodes
Nodos on-prem unidos a un control plane de EKS. Cobro por vCPU-hora y restricciones de CNI.
Candidato para Labs 6–7.

## Jetson Orin Nano
SoC con memoria unificada CPU/GPU para edge. Sin DCGM ni MIG y con otra configuración de device plugin:
no representa la inferencia de datacenter.

## Conexión con el objetivo
El device plugin decide cuántas cargas "caben" en un nodo desde la vista del scheduler; si esa cuenta
no refleja la VRAM real, la densidad aparente es falsa (ver [05](05-gpu-sharing.md)).
