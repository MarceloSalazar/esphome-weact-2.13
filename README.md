# esphome-weact-2.13

Prueba de concepto: integración de un módulo ESP8266 y una pantalla
e-ink WeAct 2.13" tricolor (Black/White/Red) con Home Assistant, usando
ESPHome.

El dispositivo muestra, actualizando cada 5 minutos (con un primer
refresco casi inmediato al arrancar, sin esperar los 5 minutos):

```
dd/mm/aaaa - hh:mm
Bateria: xx.x% xx.xV [+,-]xxxxW
Consumo Red: xxxxW
```

Las unidades (`%`, `V`, `W`) se dibujan en rojo; el resto del texto en
negro. El signo de la potencia de batería indica `+` cargando, `-`
descargando.

Datos leídos de Home Assistant (instalación Victron ESS/MultiPlus II +
Shelly EM del proyecto [HEMS](https://github.com/MarceloSalazar/hems)):

| Campo | Entidad de Home Assistant |
|---|---|
| Batería % | `sensor.victron_system_battery_soc` |
| Batería V | `sensor.victron_system_battery_voltage` |
| Batería W (signo: + carga, - descarga) | `sensor.garage_settings_battery_power_avg_5min` (helper *Statistics*, media lineal 5 min sobre `sensor.victron_system_battery_power`) |
| Consumo Red W | `sensor.entrance_shellyem_c45bbee19932_channel_1_consumo_vivienda_avg_5min` (helper *Statistics*, media lineal 5 min sobre el Shelly EM de acometida) |

Los dos sensores de media de 5 minutos son *Helpers* creados desde la UI
de Home Assistant (Ajustes → Dispositivos y Servicios → Ayudantes →
Estadística), no YAML — así no se toca la configuración de producción
de HEMS.

### Notas de diseño (gotchas no obvios)

- **Refresco inmediato sin colisionar con el periódico.** ESPHome
  dispara la primera ejecución de cualquier `interval:`/`update_interval`
  casi de inmediato al arrancar (no espera el intervalo completo). Como
  eso competía con el refresco inmediato al llegar los primeros datos
  de HA (el automático ganaba la carrera y dejaba la pantalla ocupada
  ~35s, justo cuando llegaban los datos reales), el `interval:` que
  dispara el refresco periódico usa `startup_delay` igual al propio
  intervalo (5 min) para que su primer disparo nunca compita con el
  refresco inmediato de `refresh_when_ready` (que gana la carrera en
  <1s tras arrancar, en cuanto SOC/voltaje/potencia/consumo/hora están
  listos).
- **`get_text_bounds` con alineación LEFT.** Para pintar unidades en
  otro color dentro de la misma línea hace falta medir cada tramo de
  texto y concatenar posiciones. Con `TextAlign::LEFT`, el `x1` que
  devuelve `get_text_bounds` ya es la coordenada x absoluta que se le
  pasó (no un offset relativo) — sumarlo de nuevo a `x` duplica el
  avance y descoloca el texto hacia la derecha.

Diseño completo: [docs/superpowers/specs/2026-09-17-esphome-weact-display-design.md](docs/superpowers/specs/2026-09-17-esphome-weact-display-design.md).

## Hardware

- **ESP8266** (NodeMCU, chip ESP8266EX, 4MB flash, USB-serie CP2102).
- **Pantalla e-ink WeAct 2.13" B&W&R** (controlador SSD1680, 122×250 px,
  tricolor). [Repositorio del fabricante](https://github.com/WeActStudio/WeActStudio.EpaperModule).

### Wiring (bus SPI hardware)

El módulo WeAct etiqueta los pines SPI con nombres de I2C (`SCL`/`SDA`)
aunque el bus es SPI, no I2C: `SCL` = SCK (clock), `SDA` = MOSI (data).

| Pin en el módulo (silkscreen) | Señal SPI | GPIO ESP8266 | Pin NodeMCU |
|---|---|---|---|
| CS   | CS   | GPIO15 | D8 |
| SCL  | SCK  | GPIO14 | D5 |
| SDA  | MOSI | GPIO13 | D7 |
| BUSY | BUSY | GPIO16 | D0 |
| RES  | RST  | GPIO5  | D1 |
| D/C  | DC   | GPIO4  | D2 |
| VCC  | VCC  | 3V3 | 3V3 |
| GND  | GND  | GND | GND |

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

# Uso habitual (primer flasheo, por USB):
.venv/bin/esphome compile weact-display.yaml
.venv/bin/esphome upload weact-display.yaml --device /dev/ttyUSB0
.venv/bin/esphome logs weact-display.yaml --device /dev/ttyUSB0
# o, para compilar+flashear+ver logs en un solo paso:
.venv/bin/esphome run weact-display.yaml --device /dev/ttyUSB0

# Una vez el dispositivo está en la red, se puede actualizar/depurar
# por OTA sin USB, usando su IP en vez del puerto serie:
.venv/bin/esphome upload weact-display.yaml --device 192.168.2.211
.venv/bin/esphome logs weact-display.yaml --device 192.168.2.211
```

Si el comando falla con un error de permisos en `/dev/ttyUSB0`, tu
usuario necesita pertenecer al grupo `dialout`:

```bash
sudo usermod -aG dialout $USER
# cierra sesión y vuelve a entrar (o reinicia VSCode del todo)
```

## Secretos

Copia `secrets.yaml.example` a `secrets.yaml` (ignorado por git) y
completa los valores reales: contraseña WiFi y `api_encryption_key`
(clave base64 de 32 bytes; se usa tanto para la API nativa de ESPHome
como para cifrar las actualizaciones OTA — genera una nueva con
`python3 -c "import secrets, base64; print(base64.b64encode(secrets.token_bytes(32)).decode())"`).

## Estado del proyecto

Seguimiento por fases en los [issues de GitHub](https://github.com/MarceloSalazar/esphome-weact-2.13/issues)
de este repositorio.

- [x] Entorno (PlatformIO + ESPHome) instalado y funcionando
- [x] ESP8266 detectado por USB
- [x] Firmware básico compilado y flasheado
- [x] Logs legibles por el monitor serie
- [x] Wiring físico del panel e-ink
- [x] "Hola mundo!" en el panel e-ink
- [x] Conexión WiFi (`LMT`, IP estática) + OTA (cifrado)
- [x] Integración con Home Assistant (batería + consumo de la vivienda)
