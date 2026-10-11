# Instrucciones para Claude Code

Este repo documenta un roadmap de AI Infrastructure (GPU + inferencia). Contexto completo:

@CONTEXTO.md

## Reglas
- Responder en español, directo y técnicamente preciso.
- Este repo se trabaja desde otra máquina (macOS). El home lab (k3s) se accede por SSH sin contraseña:
  `ssh esleirter@192.168.0.165`. En sesiones no interactivas `KUBECONFIG` no está en el entorno:
  usar `KUBECONFIG=/etc/rancher/k3s/k3s.yaml kubectl ...`. `sudo` pide contraseña (pedírsela al usuario
  con `! ssh -t ...`). Puedes usar `kubectl`, `helm`, `nvidia-smi` y `docker` por SSH para leer el
  estado real, pero **no modifiques el cluster sin confirmación**.
- Al terminar un lab: crear/actualizar `labs/lab-XX-*/README.md`, agregar errores a `errores.md`,
  y actualizar `README.md` y `CONTEXTO.md` (estado, hallazgos, próximos pasos, fecha).
- En `teoria/`, no agregar contenido que no se haya estudiado o comprobado; marcar con
  `> TODO:` lo que falte.
- Commits pequeños y descriptivos en español.
