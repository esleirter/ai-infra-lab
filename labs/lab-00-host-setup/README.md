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

## Comandos usados (recuperados de `~/.bash_history`)

### Driver
```bash
sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt update
ubuntu-drivers devices
sudo ubuntu-drivers install nvidia:595
sudo reboot
```

### Kernel HWE (error #1: la red RTL8125 se cayó con 5.15.0-186)
```bash
sudo apt install --install-recommends linux-generic-hwe-22.04   # → 6.8.0-136
sudo reboot
sudo ubuntu-drivers install nvidia:595
```

### DKMS con GCC 12 (error #2)
```bash
tail -30 /var/lib/dkms/nvidia/595.84/build/make.log
sudo apt install gcc-12
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-11 11
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-12 12
sudo update-alternatives --install /usr/bin/cc cc /usr/bin/gcc-12 12
sudo update-alternatives --set gcc /usr/bin/gcc-12
sudo update-alternatives --set cc /usr/bin/gcc-12
sudo dkms install nvidia/595.84 -k 6.8.0-136-generic
dkms status
sudo reboot
nvidia-smi
```

### NVIDIA Container Toolkit (error #3: `daemon.json` vacío)
```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update
export NVIDIA_CONTAINER_TOOLKIT_VERSION=1.19.1-1
echo '{}' | sudo tee /etc/docker/daemon.json      # daemon.json existía con 0 bytes
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
docker run --rm --gpus all ubuntu:22.04 nvidia-smi
```
