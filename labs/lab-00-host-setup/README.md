# Lab 0 — Preparación del host

**Fecha:** 3–4 de agosto de 2026 · **Teoría relacionada:** [03 — Container runtime](../../teoria/03-container-runtime.md)

## Objetivo
Dejar la máquina con driver NVIDIA, Docker y NVIDIA Container Toolkit funcionando.

## Contexto
Dual boot con Linux Mint 21 (base Ubuntu 22.04) en una Gigabyte B550-AORUS con GTX 1650 SUPER.

## Problemas y solución

### 1. Red rota después de instalar el driver
- **Síntoma:** desaparece la red (NIC Realtek RTL8125).
- **Causa:** el kernel 5.15.0-186 tenía incompleto el paquete de módulos.
- **Solución:** arrancar con el kernel HWE 6.8.0-136, que ya estaba instalado.

### 2. DKMS no compila el módulo NVIDIA
- **Síntoma:** `unrecognized command-line option '-ftrivial-auto-var-init=zero'`
- **Causa:** el kernel 6.8 fue compilado con GCC 12; el GCC activo era 11.4.
- **Solución:**
  ```bash
  sudo apt install gcc-12
  sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-12 12
  sudo update-alternatives --config gcc
  # reinstalar/recompilar el driver para que DKMS construya el módulo
  ```
- Nota: el PPA `graphics-drivers` estaba agregado, pero no era la causa.

### 3. `nvidia-ctk runtime configure` falla con EOF
- **Causa:** `/etc/docker/daemon.json` existía pero vacío (0 bytes); no es JSON válido.
- **Solución:**
  ```bash
  echo '{}' | sudo tee /etc/docker/daemon.json
  sudo nvidia-ctk runtime configure --runtime=docker
  sudo systemctl restart docker
  ```

## Resultado
| Componente | Versión |
|---|---|
| Kernel | 6.8.0-136 (HWE) |
| Driver | 595.84 (techo CUDA 13.2) |
| Docker | 27.1.1 |
| NVIDIA Container Toolkit | 1.19.1 |

## Lección
Antes de culpar al driver, revisar la cadena completa: kernel → headers/módulos → compilador → DKMS.
Un update de kernel puede volver a romper el módulo; verificar `nvidia-smi` después de cada update.

> TODO: completar con los comandos exactos de instalación del driver y del repo del toolkit
> (revisar historial: `history | grep -E "nvidia|apt"`).
