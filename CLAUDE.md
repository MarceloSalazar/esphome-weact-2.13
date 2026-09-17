# CLAUDE.md

Convenciones para trabajar en este repositorio con Claude Code.

## Naturaleza del proyecto

PoC de hardware embebido: ESP8266 + pantalla e-ink WeAct 2.13" B&W&R +
Home Assistant, vía ESPHome. Prioriza validar que la integración
funciona end-to-end sobre la robustez de producción. Ver el diseño
completo en `docs/superpowers/specs/2026-09-17-esphome-weact-display-design.md`.

## Entorno

- Todo el tooling (ESPHome, PlatformIO) vive en `.venv/`, con Python
  3.12 gestionado por `uv` (el Python 3.10 del sistema no alcanza el
  mínimo de ESPHome ≥2026.x). No asumas que `esphome`/`platformio`
  están en el PATH del sistema — usa siempre `.venv/bin/esphome`.
- El dispositivo se conecta en `/dev/ttyUSB0` (chip USB-serie CP2102).
  Si un comando falla con error de permisos, el usuario necesita estar
  en el grupo `dialout` (`sudo usermod -aG dialout $USER` + reiniciar
  sesión/VSCode). Como workaround dentro de una misma sesión de shell
  que aún no heredó el grupo, se puede anteponer `sg dialout -c '...'`.
- Un solo archivo de configuración ESPHome: `weact-display.yaml`. No
  introducir `packages:` ni dividir en más archivos a menos que el
  proyecto crezca a más de un dispositivo.

## Secretos

- `secrets.yaml` nunca se commitea (está en `.gitignore`). Los cambios
  a claves usadas (`!secret ...`) deben reflejarse también en
  `secrets.yaml.example` con valores ficticios.

## Hardware

- Pinout fijo del wiring SPI documentado en el README y en el spec —
  no cambiarlo sin verificar antes con el usuario, ya que implica
  recablear la placa físicamente.
- La pantalla es tricolor (B&W&R) y **no soporta refresco parcial
  confiable** — cualquier cambio en la lógica de refresco debe seguir
  minimizando la frecuencia de refrescos completos (ver estrategia de
  `throttle` + detección de cambio de valor en el spec).

## Seguimiento de trabajo

- Todas las tareas y bugs se registran como issues en el repositorio
  de GitHub (`MarceloSalazar/esphome-weact-2.13`), no en TODOs sueltos
  del código ni en documentos aparte.
