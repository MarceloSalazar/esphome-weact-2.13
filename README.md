# esphome-weact-2.13

Prueba de concepto: integración de un módulo ESP8266 y una pantalla
e-ink WeAct 2.13" tricolor (Black/White/Red) con Home Assistant, usando
ESPHome.

El dispositivo final muestra el consumo eléctrico de la vivienda (en
vatios), leído desde un sensor existente de Home Assistant, actualizando
la pantalla al menos cada minuto.

Diseño completo: [docs/superpowers/specs/2026-09-17-esphome-weact-display-design.md](docs/superpowers/specs/2026-09-17-esphome-weact-display-design.md).

## Hardware

- **ESP8266** (NodeMCU, chip ESP8266EX, 4MB flash, USB-serie CP2102).
- **Pantalla e-ink WeAct 2.13" B&W&R** (controlador SSD1680, 122×250 px,
  tricolor). [Repositorio del fabricante](https://github.com/WeActStudio/WeActStudio.EpaperModule).

### Wiring (bus SPI hardware)

| Señal e-ink | GPIO ESP8266 | Pin NodeMCU |
|---|---|---|
| CS   | GPIO15 | D8 |
| SCK  | GPIO14 | D5 |
| MOSI | GPIO13 | D7 |
| BUSY | GPIO16 | D0 |
| RST  | GPIO5  | D1 |
| DC   | GPIO4  | D2 |
| VCC  | 3V3 | 3V3 |
| GND  | GND | GND |

## Software

- [ESPHome](https://esphome.io/) 2026.9.0+ (necesita Python ≥3.12 —
  imprescindible para el componente `epaper_spi` que soporta esta
  pantalla tricolor).
- PlatformIO (lo instala ESPHome automáticamente como dependencia).

## Entorno de desarrollo

Todo corre dentro de un entorno virtual Python en `.venv/`, gestionado
con [`uv`](https://docs.astral.sh/uv/) (necesario porque el Python 3.10
por defecto del sistema no cumple el mínimo de ESPHome):

```bash
# Primera vez / si falta Python 3.12:
uv python install 3.12
uv venv --python 3.12 .venv
uv pip install --python .venv/bin/python esphome

# Uso habitual:
.venv/bin/esphome compile weact-display.yaml
.venv/bin/esphome upload weact-display.yaml --device /dev/ttyUSB0
.venv/bin/esphome logs weact-display.yaml --device /dev/ttyUSB0
# o, para compilar+flashear+ver logs en un solo paso:
.venv/bin/esphome run weact-display.yaml --device /dev/ttyUSB0
```

Si el comando falla con un error de permisos en `/dev/ttyUSB0`, tu
usuario necesita pertenecer al grupo `dialout`:

```bash
sudo usermod -aG dialout $USER
# cierra sesión y vuelve a entrar (o reinicia VSCode del todo)
```

## Secretos

Copia `secrets.yaml.example` a `secrets.yaml` (ignorado por git) y
completa los valores reales (contraseña WiFi, etc.).

## Estado del proyecto

Seguimiento por fases en los [issues de GitHub](https://github.com/MarceloSalazar/esphome-weact-2.13/issues)
de este repositorio.

- [x] Entorno (PlatformIO + ESPHome) instalado y funcionando
- [x] ESP8266 detectado por USB
- [x] Firmware básico compilado y flasheado
- [x] Logs legibles por el monitor serie
- [ ] Wiring físico del panel e-ink
- [ ] "Hola mundo!" en el panel e-ink
- [ ] Conexión WiFi (`LMx`)
- [ ] Integración con Home Assistant (consumo de la vivienda en vatios)
