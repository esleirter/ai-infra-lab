# Lab 2 — k3s + device plugin + Time-Slicing

**Fechas:** 4 de agosto (k3s + device plugin) y 23 de agosto de 2026 (Time-Slicing)
**Teoría relacionada:** [03 — Container runtime](../../teoria/03-container-runtime.md), [04 — Kubernetes y GPU](../../teoria/04-kubernetes-gpu.md), [05 — Compartir GPU](../../teoria/05-gpu-sharing.md)

---

## Parte A — k3s con GPU

### 1. Config antes de instalar
El archivo de config se lee sin importar cómo se instale k3s, y el script no lo gestiona: conviene
crearlo **antes**.

```bash
sudo mkdir -p /etc/rancher/k3s
sudo tee /etc/rancher/k3s/config.yaml <<'CFG'
write-kubeconfig-mode: "0644"
node-label:
  - "gpu=true"
  - "gpu.model=gtx1650super"
CFG
```
- `write-kubeconfig-mode` evita copiar el kubeconfig con sudo.
- Las labels sirven para `nodeSelector` (misma mecánica que con Karpenter en EKS).

### 2. Instalar
```bash
curl -sfL https://get.k3s.io | sh -
echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl get nodes --show-labels
```
Versión: v1.36.2+k3s1, containerd 2.3.2-k3s2. El script deja k3s como servicio systemd y genera
`k3s-uninstall.sh` (el binario suelto corre en primer plano y no sirve para algo permanente).

### 3. Lo que k3s hizo solo
Detectó el NVIDIA Container Toolkit del Lab 0/1, registró el runtime y creó el RuntimeClass `nvidia`:

```bash
sudo grep -A6 nvidia /var/lib/rancher/k3s/agent/etc/containerd/config.toml
```
```toml
[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.'nvidia']
  runtime_type = "io.containerd.runc.v2"
[...runtimes.'nvidia'.options]
  BinaryName = "/usr/bin/nvidia-container-runtime"
  SystemdCgroup = true
```
`runtime_type` es runc, pero `BinaryName` apunta a NVIDIA: al final *es* runc, solo cambia quién
prepara el OCI spec antes. (Los RuntimeClass `wasmtime`, `spin`, `crun` son de WebAssembly; k3s los
declara siempre.)

### 4. Device plugin (manifiesto crudo, no Helm)
```bash
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.17.1/deployments/static/nvidia-device-plugin.yml
kubectl -n kube-system logs -l name=nvidia-device-plugin-ds
kubectl describe node b550-aorus | grep -A10 "Capacity:"   # nvidia.com/gpu: 1
```

### Errores de la Parte A

**Error 1 — Las labels no se aplicaron.** La primera ejecución arrancó k3s antes de que existiera el
`config.yaml`; al repetir el script: `No change detected so skipping service start`.
→ `sudo systemctl restart k3s`

**Error 2 — El plugin no ve la GPU.** Pod `1/1 Running`, pero en logs:
```
E factory.go:112] Incompatible strategy detected auto
E factory.go:113] If this is a GPU node, did you configure the NVIDIA Container Toolkit?
I main.go:381] No devices found. Waiting indefinitely.
```
El mensaje despista: el toolkit sí estaba. El pod del plugin corría con runc y no tenía `/dev/nvidia*`;
necesita ver la GPU para anunciarla.
```bash
kubectl -n kube-system patch daemonset nvidia-device-plugin-daemonset --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/runtimeClassName","value":"nvidia"}]'
```

**Error 3 — Recurso sin RuntimeClass y RuntimeClass sin recurso.**

| Pod | RuntimeClass | Recurso | Resultado |
|---|---|---|---|
| test-con | nvidia | — | Runtime activo, pero sin saber qué inyectar |
| test-gpu | runc | gpu: 1 | Recurso reservado, pero runc no inyecta nada |
| **gpu-test** | nvidia | gpu: 1 | ✅ Funciona |

- `nvidia.com/gpu: 1` = **la solicitud**: el scheduler asigna y el plugin inyecta `NVIDIA_VISIBLE_DEVICES`.
- `runtimeClassName: nvidia` = **quién ejecuta**: el runtime lee esa variable e inyecta el dispositivo.

**Error 4 — `nvidia-smi` no está en el PATH.** Con RuntimeClass y recurso correctos, el pod falla al crear:
```
OCI runtime create failed: runc create failed:
exec: "nvidia-smi": executable file not found in $PATH
```
El binario `nvidia-smi` lo inyecta el runtime NVIDIA en un hook de prearranque, pero el `command` del
pod se resuelve contra el PATH de la imagen **antes** de ese hook.
→ Usar ruta absoluta (`/usr/bin/nvidia-smi`) o, mejor, una imagen CUDA (lo que se haría en producción).

Manifiesto: [`manifests/gpu-test.yaml`](manifests/gpu-test.yaml)

**Opcional — `nvidia` como runtime por defecto.** k3s regenera `config.toml` en cada arranque, así que
se edita el `.tmpl`:
```bash
sudo cp /var/lib/rancher/k3s/agent/etc/containerd/config.toml \
        /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl
sudo sed -i 's/default_runtime_name = "runc"/default_runtime_name = "nvidia"/' \
        /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl
sudo systemctl restart k3s
```
**Verificado el 11 de octubre de 2026: nunca se aplicó.** No existe `config.toml.tmpl` (solo
`config.toml`, regenerado ese día al arrancar k3s) y `grep default_runtime` en `config.toml` no devuelve
nada: containerd usa su default, `runc`. Por eso todos los pods GPU, incluido el device plugin, declaran
`runtimeClassName: nvidia`. Es una decisión consciente: los pods sin GPU no pasan por el runtime de NVIDIA.

---

## Parte B — Time-Slicing

### Detectar cómo se instaló el plugin
```bash
kubectl -n kube-system get ds nvidia-device-plugin-daemonset -o yaml | grep -A5 -E "image:|env:|volumeMounts:"
```
Manifiesto crudo: un solo env (`FAIL_ON_INIT_ERROR`), un solo volumeMount y sin sidecar
`config-manager`. **Eso define cómo se configura.**

### 1. ConfigMap
[`manifests/time-slicing-cm.yaml`](manifests/time-slicing-cm.yaml) — key **`config.yaml`**, `replicas: 4`.
```bash
kubectl apply -f manifests/time-slicing-cm.yaml
```

### 2. Patch al DaemonSet (volumen + mount + `CONFIG_FILE`)
```bash
kubectl -n kube-system patch ds nvidia-device-plugin-daemonset --type='json' -p='[
  {"op":"add","path":"/spec/template/spec/volumes/-","value":{"name":"time-slicing-config","configMap":{"name":"time-slicing-config"}}},
  {"op":"add","path":"/spec/template/spec/containers/0/volumeMounts/-","value":{"name":"time-slicing-config","mountPath":"/etc/nvidia-device-plugin"}},
  {"op":"add","path":"/spec/template/spec/containers/0/env/-","value":{"name":"CONFIG_FILE","value":"/etc/nvidia-device-plugin/config.yaml"}}
]'
```

### 3. Verificar
```bash
kubectl -n kube-system rollout status ds/nvidia-device-plugin-daemonset
kubectl describe node b550-aorus | grep -A8 "Allocatable:"   # nvidia.com/gpu: 4
```
`nvidia-smi` sigue mostrando 4 GB y 20 SMs: Kubernetes cree que hay 4 GPUs, el hardware es el mismo.

### Error de la Parte B — key `any`
Primer intento con el key `any` (convención de Helm) → `CrashLoopBackOff`:
`open /etc/nvidia-device-plugin/config.yaml: no such file or directory`.

Al montar un ConfigMap como volumen, **cada key se convierte en un nombre de archivo**: se montó
`/etc/nvidia-device-plugin/any`. Helm puede usar `any` porque su sidecar `config-manager` lee el
ConfigMap por API y nunca monta el archivo.

### 4. Experimento: OOM concurrente
`hog-1` y `hog-2`, cada uno reservando 2.2 GB y pidiendo `nvidia.com/gpu: 1`.

- El scheduler **admitió ambos** (ve 4 GPUs virtuales).
- `hog-1` reservó 2.20 GB.
- `hog-2` murió con `CUDA out of memory`, reportando `total capacity of 3.63 GiB` y viendo el proceso
  del otro pod.

Manifiestos: [`manifests/hog-1.yaml`](manifests/hog-1.yaml) y [`manifests/hog-2.yaml`](manifests/hog-2.yaml)
(idénticos salvo el nombre; reservan 2.2 GB con `torch.empty` en float16). El
[`hog.yaml` del Lab 4](../lab-04-helm-mps/hog.yaml) es el mismo pod, parametrizable con la variable `GB`.

### Conclusión
Time-Slicing reparte **tiempo de cómputo, no memoria**. Kubernetes contó recursos que no existen y el
driver nunca se enteró. `nvidia.com/gpu` funciona como contador de tickets, no como garantía de VRAM.
Con MIG, cada instancia tendría su VRAM garantizada y el segundo pod no se habría programado.
En producción multi-tenant con modelos de tamaño arbitrario, esto es una bomba de tiempo.

## Lecciones
1. Un pod GPU necesita la solicitud **y** el runtime; el pod del device plugin también.
2. Al montar un ConfigMap como volumen, el key es el filename.
3. El método de instalación determina el método de configuración: verificar antes de configurar.
4. Time-Slicing no aísla VRAM.
