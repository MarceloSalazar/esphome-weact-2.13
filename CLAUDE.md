# CLAUDE.md

Conventions for working in this repository with Claude Code.

## Project nature

Embedded hardware PoC: ESP8266 + WeAct 2.13" B&W&R e-ink display +
Home Assistant, via ESPHome. Prioritizes validating that the
integration works end-to-end over production-grade robustness. See
the full design in
`docs/superpowers/specs/2026-09-17-esphome-weact-display-design.md`.

## Environment

- All tooling (ESPHome, PlatformIO) lives in `.venv/`, with Python
  3.12 managed by `uv` (the system's Python 3.10 doesn't meet
  ESPHome's ≥2026.x minimum). Don't assume `esphome`/`platformio` are
  on the system PATH — always use `.venv/bin/esphome`.
- The device connects at `/dev/ttyUSB0` (CP2102 USB-serial chip). If a
  command fails with a permissions error, the user needs to be in the
  `dialout` group (`sudo usermod -aG dialout $USER` + restart the
  session/VSCode). As a workaround within a shell session that hasn't
  picked up the group yet, prefix commands with `sg dialout -c '...'`.
- A single ESPHome config file: `weact-display.yaml`. Don't introduce
  `packages:` or split it into more files unless the project grows to
  more than one device.

## Secrets

- `secrets.yaml` is never committed (it's in `.gitignore`). Changes to
  keys used (`!secret ...`) must also be reflected in
  `secrets.yaml.example` with placeholder values.
- Network settings that reveal the deployer's home network (WiFi SSID,
  static IP, gateway) are sourced via `!secret` too, not hardcoded in
  `weact-display.yaml` — this repo is public, so anything specific to
  one person's install belongs in their local `secrets.yaml`, not in
  git.

## Hardware

- The SPI wiring pinout is documented in the README and in the spec —
  don't change it without checking with the user first, since it means
  physically rewiring the board.
- The display is tricolor (B&W&R) and **does not support reliable
  partial refresh** — any change to the refresh logic should keep
  minimizing how often a full refresh happens (see the throttle +
  change-detection strategy in the spec).

## Work tracking

- All tasks and bugs are tracked as issues in the GitHub repository
  (`MarceloSalazar/esphome-weact-2.13`), not as loose TODOs in code or
  in separate documents.
