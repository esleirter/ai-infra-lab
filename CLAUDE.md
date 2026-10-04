# Instrucciones para Claude Code

Este repo documenta un roadmap de AI Infrastructure (GPU + inferencia). Contexto completo:

@CONTEXTO.md

## Reglas
- Responder en español, directo y técnicamente preciso.
- Este repo corre en la misma máquina del home lab (k3s local). Puedes usar `kubectl`, `nvidia-smi`
  y `docker` para leer el estado real, pero **no modifiques el cluster sin confirmación**.
- Al terminar un lab: crear/actualizar `labs/lab-XX-*/README.md`, agregar errores a `errores.md`,
  y actualizar `README.md` y `CONTEXTO.md` (estado, hallazgos, próximos pasos, fecha).
- En `teoria/`, no agregar contenido que no se haya estudiado o comprobado; marcar con
  `> TODO:` lo que falte.
- Commits pequeños y descriptivos en español.
