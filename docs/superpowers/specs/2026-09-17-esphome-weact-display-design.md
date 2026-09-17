# Diseño: ESP8266 + WeAct 2.13" e-ink (B&W&R) + Home Assistant

Fecha: 2026-09-17
Estado: aprobado por el usuario, pendiente de implementación

## Objetivo

Probar la integración y el funcionamiento de dos componentes con Home
Assistant, usando ESPHome:

1. Un módulo ESP8266 (tipo NodeMCU) conectado por USB.
2. Un módulo e-ink WeAct 2.13" tricolor (Black/White/Red).

El resultado final es un dispositivo ESPHome que muestra el consumo
eléctrico de la vivienda (en vatios) leído desde un sensor existente de
Home Assistant, actualizando la pantalla al menos cada minuto.

Este es un proyecto de prueba de concepto (PoC), no un despliegue de
producción: se prioriza validar que la integración funciona end-to-end
sobre la robustez a largo plazo.

## Hardware

### ESP8266

- Placa tipo NodeMCU (modelo exacto por confirmar cuando se detecte por
  USB — ver "Decisiones pendientes").
- Framework: Arduino (vía ESPHome, plataforma `esp8266:`).
- GPIOs utilizables sin restricciones de boot: GPIO4, GPIO5, GPIO12,
  GPIO13, GPIO14, GPIO16. Se evitan GPIO0/2/15 (modo de arranque),
  GPIO1/3 (UART), GPIO6-11 (flash interno).

### Panel e-ink WeAct 2.13" B&W&R

- Controlador: SSD1680.
- Panel: GDEY0213Z98, 122×250 píxeles, tricolor (negro/blanco/rojo).
- Solo soporta refresco completo (sin refresco parcial confiable) — un
  ciclo de refresco completo tarda ~15-20s y produce parpadeo visible.
- Fuente: ejemplo oficial de WeAct para ESP8266
  (`WeActStudio.EpaperModule/Example/EpaperModuleTest_Arduino_ESP8266`)
  y hoja de datos SSD1680 incluida en el mismo repositorio
  (`Doc/SSD1680_Datasheet.pdf`).

### Wiring propuesto (SPI hardware del ESP8266)

| Señal e-ink | GPIO ESP8266 | Pin NodeMCU (referencia) |
|---|---|---|
| CS   | GPIO15 | D8 |
| SCK/CLK | GPIO14 | D5 |
| MOSI/SDA | GPIO13 | D7 |
| BUSY | GPIO16 | D0 |
| RST  | GPIO5  | D1 |
| DC   | GPIO4  | D2 |
| VCC  | 3V3 | 3V3 |
| GND  | GND | GND |

Se usa el bus SPI hardware (HSPI: GPIO12/13/14) para CLK/MOSI, tal como
recomienda el fabricante en su propio ejemplo — evita SPI por software
y es más fiable a la velocidad de refresco del panel. MISO (GPIO12) no
se usa (el panel no lo requiere) pero queda reservado por si el futuro
componente SPI de ESPHome lo pide en la configuración del bus.

GPIO15 (CS) requiere estar en LOW durante el arranque para no interferir
con el modo de boot; al ser una salida controlada activamente por SPI,
no supone un problema en la práctica, pero se documenta como punto de
atención si aparecen arranques erráticos.

## Software

- **ESPHome** (CLI + YAML), instalado vía `pip`/entorno virtual Python.
  Versión mínima: la que incluya el componente `epaper_spi` con el
  modelo `weact-2.13in-3c` — disponible desde la release **2026.9.0**
  (publicada 2026-09-16). No usar versiones anteriores a febrero 2026.
- **PlatformIO**: usado internamente por ESPHome para compilar el
  firmware; no se escribe código Arduino/PlatformIO a mano. Se instala
  como dependencia del entorno, y la extensión de VSCode de PlatformIO
  se usa solo como apoyo (monitor serie, exploración del build), no
  como flujo de compilación principal.
- **Extensión ESPHome para VSCode**: resaltado de sintaxis y validación
  de YAML.
- Encoding de color de ESPHome para el display: `platform: epaper_spi`,
  `model: weact-2.13in-3c` (NO usar `waveshare_epaper`, que no soporta
  esta variante tricolor de 2.13").

## Estructura del repositorio

```
esphome-weact-2.13/
├── README.md              # documentación principal del proyecto
├── CLAUDE.md               # convenciones para trabajar con Claude Code
├── .gitignore               # excluye secrets.yaml, .esphome/, .pio/
├── secrets.yaml             # WiFi password, etc. (NO se commitea)
├── secrets.yaml.example     # plantilla de secrets.yaml, sí se commitea
└── weact-display.yaml       # configuración ESPHome única
```

Un solo archivo YAML de ESPHome: es un único dispositivo con un único
propósito. No se introduce `packages:` ni separación en múltiples
archivos — se reconsiderará solo si el proyecto crece a más de un
dispositivo o pantalla.

## Configuración ESPHome (componentes clave)

- `esphome:` — nombre del dispositivo, `substitutions:` para valores
  configurables (ver más abajo).
- `esp8266:` — `board:` (a confirmar), `framework: arduino`.
- `spi:` — bus SPI hardware, `clk_pin: GPIO14`, `mosi_pin: GPIO13`.
- `display:`
  - `platform: epaper_spi`
  - `model: weact-2.13in-3c`
  - `cs_pin`, `dc_pin`, `busy_pin`, `reset_pin` según tabla de wiring.
  - `update_interval:` controlado por substitution (ver estrategia de
    refresco).
  - `full_update_every:` para forzar limpieza periódica y evitar
    ghosting (valor exacto a ajustar empíricamente, punto de partida:
    cada 30 refrescos).
- `wifi:` — `ssid: "LMx"`, `password: !secret wifi_password`.
- `api:` — API nativa de ESPHome; Home Assistant descubre el
  dispositivo por mDNS, sin necesidad de token manual.
- `logger:` — salida por UART (hardware_uart: UART0), para depuración
  por puerto serie.
- `sensor:` — `platform: homeassistant`, `entity_id:` (placeholder
  hasta que se confirme el sensor real de consumo de la vivienda),
  con un filtro `throttle` para no redibujar más de una vez por
  intervalo configurado.

### Substitutions propuestas

```yaml
substitutions:
  display_refresh_interval: "60s"   # "configurable a futuro", punto 1
  power_entity_id: "sensor.PENDIENTE_consumo_vivienda"
```

## Flujo de datos

1. El sensor de potencia ya existe en Home Assistant (integración de
   medición de consumo, aún sin identificar — ver decisiones
   pendientes).
2. HA empuja el valor al ESP8266 vía la API nativa de ESPHome cada vez
   que el estado del sensor cambia (push, no polling).
3. Un filtro `throttle: ${display_refresh_interval}` en el sensor
   limita cuántas veces por minuto se dispara la actualización de
   pantalla.
4. Solo si el valor mostrado cambia (redondeo a vatios enteros, por
   ejemplo) se dispara un refresco completo del e-ink.
5. El e-ink permanece con la última imagen dibujada entre refrescos
   (comportamiento nativo de e-ink, no requiere lógica adicional).

## Manejo de errores

- **WiFi caído**: reconexión automática nativa de ESPHome. No se
  sobreescribe la última imagen válida en el e-ink.
- **API/HA caída**: reconexión automática nativa de `api:`. El filtro
  del sensor ignora valores `NaN`/desconocidos, evitando redibujar con
  datos basura o "unavailable".
- **Ghosting del panel**: mitigado con `full_update_every`.
- **Fallo de compilación/flash**: se valida en cada fase con
  `esphome compile` y `esphome upload` antes de dar la fase por
  cerrada (ver plan de fases).

## Pruebas y validación

No hay tests automatizados tradicionales en un proyecto ESPHome — la
validación es por fases, cada una con su propio criterio de "hecho":

1. Entorno (PlatformIO + ESPHome instalados y funcionando).
2. ESP8266 detectado por USB (puerto serie visible).
3. Firmware básico compilado y flasheado con éxito.
4. Logs legibles por el monitor serie de ESPHome.
5. Wiring físico documentado y validado con continuidad/multímetro si
   hace falta.
6. Texto "Hola mundo!" visible en el panel e-ink.
7. Dispositivo conectado a la red WiFi `LMx`.
8. Valor de consumo de HA visible en el panel, actualizando según la
   estrategia de refresco definida.

Cada fase se registra como un issue de GitHub en el repositorio
`MarceloSalazar/esphome-weact-2.13`, con su criterio de aceptación.

## Gestión de secretos

- `secrets.yaml` (gitignored) contiene `wifi_password` y cualquier
  otro valor sensible.
- `secrets.yaml.example` se commitea como plantilla, con valores
  ficticios.
- `.gitignore` excluye además `.esphome/` y `.pio/` (directorios de
  build generados por ESPHome/PlatformIO).

## Decisiones pendientes (deliberadamente diferidas)

- **Modelo exacto de la placa ESP8266** (`board:` en la config):
  pendiente de confirmar cuando el dispositivo se detecte por USB.
  Troubleshooting de detección en curso.
- **`entity_id` del sensor de consumo de la vivienda en Home
  Assistant**: el usuario indicó que no es prioritario ahora; se
  resolverá antes de implementar el paso 10 (integración final con
  HA). Hasta entonces, el YAML usa un placeholder claramente marcado.
- **Valor exacto de `full_update_every`**: se ajustará empíricamente
  una vez el panel esté operativo, observando el ghosting real.

## Fuera de alcance

- Gestión de energía/batería (el dispositivo se alimenta por USB de
  forma permanente).
- Refresco parcial del panel (no soportado de forma fiable por el
  controlador SSD1680 en modo tricolor).
- Múltiples pantallas o dispositivos adicionales.
- Autenticación/token manual de Home Assistant (se usa descubrimiento
  nativo vía API de ESPHome).
