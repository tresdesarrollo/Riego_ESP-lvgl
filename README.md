# Riego_ESP-lvgl
Controlador de riego con 4 programas, 8 zonas e interfaz con pantalla táctil, hecho con ESPHome y LVGL para la placa  Guition JC4827W543C, (ESP32-S3, pantalla de 4,3" 480×272).

Funciona **sin necesidad de Home Assistant**: todo se maneja desde la pantalla. Si lo conectas a Home Assistant, además aparecen todas las entidades.

> **Código abierto.** Este proyecto es de código abierto y existe gracias a que ESPHome y LVGL también lo son. Consulta el apartado [Licencia](#licencia).

---

## Qué hace

- **8 zonas** de riego con duración (0–60 min) y nombre editables desde la pantalla.
- **4 programas** con hora de inicio cada 15 minutos. Cada programa riega las 8 zonas en orden, con sus duraciones.
- **Multiplicador de tiempo** (0–120 %) para alargar o acortar todos los riegos.
- **Saltar días** (0–10): riega un día de cada N.
- **Lógica de lluvia** con un sensor binario: reduce el riego o lo salta según el tiempo que ha llovido en las últimas 24 horas. Se configura desde la propia pantalla y se puede desactivar con un botón.
- **Modo manual**: tocar una zona o el botón "1 Ronda" riega siempre, llueva o no (pensado para pruebas y reparaciones).
- Hora por internet (NTP) y reloj con batería (PCF8563, a 3,3 V) que la conserva si se corta la luz. Necesita WiFi para obtener la hora (vale compartir el del móvil).
- Brillo y tiempo de iluminación de la pantalla ajustables.
- Hotspot WiFi de emergencia con código QR en pantalla.

## Capturas

Fotos reales de la pantalla del controlador.

| | |
|:-:|:-:|
| ![Inicio sin lluvia](docs/img/inicio-sol.jpg) | ![Inicio con lluvia](docs/img/inicio-lluvia-activo.jpg) |
| **Inicio**: sin lluvia (sol) y lógica de lluvia **ACTIVA** | **Inicio**: el sensor detecta lluvia y la lógica está **ACTIVA** |
| ![Inicio con el sensor desactivado](docs/img/inicio-lluvia-off.jpg) | ![Programas con una zona regando](docs/img/programas-zona-regando.jpg) |
| **Inicio**: sensor **OFF**. El icono sigue mostrando lo que lee el sensor, pero la lógica lo ignora | **Programas**: la zona 4 regando (en verde) |
| ![Programas con un programa desactivado](docs/img/programas-prog3-off.jpg) | ![Ajustes](docs/img/ajustes.jpg) |
| **Programas**: el programa 3 desactivado (en gris) | **Ajustes**, con el botón **Lluvia** abajo a la derecha |
| ![Página Lluvia](docs/img/pagina-lluvia.jpg) | ![Nombre de una zona](docs/img/teclado-nombre-zona.jpg) |
| **Lluvia**: activar o desactivar el sensor, umbrales y borrado del histórico | Editar el **nombre de una zona** (pulsación larga sobre su duración) |
| ![Código QR del hotspot](docs/img/qr-hotspot.jpg) | |
| **Código QR** del hotspot WiFi de emergencia. La clave de la captura (`88888888`) es de ejemplo, ESPHome exige al menos 8 caracteres. Usa la tuya | |

## Hardware

| Pieza | Conexión |
|---|---|
| Placa Guition JC4827W543C (ESP32-S3, 4 MB flash, PSRAM octal) | – |
| Pantalla táctil GT911 | I2C `bus_a`: SDA GPIO8, SCL GPIO4, interrupción GPIO3 |
| Placa de relés de 8 canales con expansor **XL9535** | I2C `bus_b`: SDA GPIO6, SCL GPIO7 (100 kHz), dirección `0x20` |
| Reloj **PCF8563** (módulo que funcione a **3,3 V**) | I2C `bus_b`, dirección `0x51` |
| Sensor de lluvia (contacto seco) | GPIO15 y GND, con pull-up interno |
| Retroiluminación | GPIO1 (PWM) |

**Reloj.** Alimenta el módulo a **3,3 V**, no a 5 V (ver el apartado «Solución de problemas»). Si no tienes reloj conectado, el sistema funciona igualmente: la hora la da internet (NTP) en cuanto hay WiFi, pero tras cada corte de luz necesitará WiFi de nuevo para recuperarla y, hasta entonces, no habrá riego automático.

Los pines del sensor de lluvia se cambian en `substitutions` (`sensor_lluvia_pin`). El resto son los de la placa y del montaje original.

**Sensor de lluvia.** Se espera un contacto que se cierra cuando llueve (normalmente abierto, NO) y la lógica ya lo invierte. Si el tuyo es normalmente cerrado (NC), cambia en el YAML `pullup` por `pulldown`, como indica el comentario junto al sensor. Colócalo **en alto y donde no le lleguen los aspersores**: el contador de lluvia suma el tiempo con el sensor activo también mientras riegas.

> ⚠️ Los relés manejan electroválvulas y, por tanto, agua y tensión. Haz el montaje eléctrico con cuidado y bajo tu responsabilidad.

## Instalación

### 1. Descarga el proyecto

Necesitas `riego-esp-lvgl.yaml`, la carpeta `fonts/` y `secrets.yaml.example`.

### 2. Fuente de iconos

Copia `materialdesignicons-webfont.ttf` en la carpeta `fonts/` (instrucciones en `fonts/README.md`). Roboto la descarga ESPHome sola.

### 3. Claves

Copia `secrets.yaml.example` como `secrets.yaml` y rellénalo:

```yaml
wifi_ssid: "NOMBRE_DE_TU_WIFI"
wifi_password: "CLAVE_DE_TU_WIFI"
api_encryption_key: "..."   # genera una con:  openssl rand -base64 32
ap_password: "cambia-esta-clave"   # mínimo 8 caracteres
```

`secrets.yaml` no se sube a GitHub. La misma clave `api_encryption_key` protege la API y las actualizaciones por WiFi (OTA).

### 4. Personaliza (opcional)

Todo lo que cambia de un montaje a otro está en el bloque `substitutions` al principio del YAML. El resto del fichero no hace falta tocarlo.

| Substitution | Para qué sirve | Por defecto |
|---|---|---|
| `nom` | Nombre del dispositivo en la red y en ESPHome (minúsculas, sin espacios ni tildes) | `riego-esp-lvgl` |
| `amigable_nom` | Nombre que ves en Home Assistant | `Riego ESP LVGL` |
| `hostname` | Texto de la barra superior de la pantalla (máx. ~15 caracteres), nombre del hotspot (`<texto> FH`) y del botón de reinicio | `Riego ESP LVGL` |
| `zona_horaria` | Zona horaria ([lista](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)) | `Europe/Madrid` |
| `sensor_lluvia_pin` | Pin del sensor de lluvia | `GPIO15` |

> Cambiar `nom` en un dispositivo que ya está dado de alta en Home Assistant hace que HA lo vea como un dispositivo nuevo. Elígelo antes del primer arranque.

### 5. Primer flasheo (por cable)

Con ESPHome instalado (dashboard de Home Assistant, o línea de comandos):

```bash
esphome run riego-esp-lvgl.yaml
```

Elige el puerto USB. **La primera vez hazlo siempre por cable.** Las siguientes veces ya puedes actualizar por WiFi (OTA).

Si usas Home Assistant, el dispositivo se descubre solo; al añadirlo te pedirá la clave `api_encryption_key`.

## Uso de la pantalla

La barra de abajo cambia entre **Inicio**, **Programas** y **Ajustes**. La página **Lluvia** se abre desde un botón en Ajustes. Tocar la pantalla la despierta cuando está apagada.

### Inicio

Icono grande de sol o lluvia según el sensor y, debajo, el estado de la lógica de lluvia (**ACTIVO** en verde o **OFF** en naranja). Mientras se riega, el icono pasa a un aspersor y se muestran la zona en curso, el tiempo restante y un indicador giratorio.

### Programas

- **A la izquierda**, los 4 programas: actívalos o desactívalos y elige su hora de inicio.
- **A la derecha**, las 8 zonas: ajusta la duración con las flechas. Una **pulsación larga** sobre el número abre el teclado para **cambiar el nombre** de la zona.
- **Tocar el botón de una zona** la riega manualmente durante su tiempo (por tanto, multiplicado por el multiplicador de tiempo) sin mirar la lluvia. Volver a tocarla la para.

### Ajustes

- Brillo y tiempo de iluminación de la pantalla.
- **X tiempo**: multiplicador de tiempo (0–120 %).
- **Saltar días**: 0 riega todos los días, 1 un día sí y otro no, etc.
- **1 Ronda**: riega las 8 zonas una vez, manualmente.
- **Reloj**: indica si la hora está sincronizada con internet y permite **sincronizarla al momento** (*Sincronizar ahora*). La hora siempre viene de internet (NTP), así que el controlador necesita WiFi, aunque sea el del móvil, al menos una vez: el reloj PCF8563 la conserva después.
- Código QR para conectar al hotspot de emergencia.
- Botón **Lluvia**: abre la configuración del sensor.

### Protección de la pantalla (anti burn-in)

Un controlador de riego pasa días con la misma imagen en pantalla. Aunque la retroiluminación se apague, el panel LCD **sigue mostrando esa imagen** (aunque nadie la vea), y eso puede acabar dejando marcas permanentes en los píxeles tras años de uso. Para evitarlo, este proyecto aplica la receta *Prevent burn-in of LCD* del [cookbook de ESPHome](https://esphome.io/cookbook/lvgl/):

- Tras el tiempo de iluminación (Ajustes), vuelve a Inicio, apaga la retroiluminación y **pausa LVGL mostrando "nieve"** (ruido aleatorio en el panel), de modo que ningún píxel se queda días con el mismo valor.
- Al **tocar la pantalla**, reanuda LVGL, redibuja la interfaz y solo entonces enciende la luz, para que no llegue a verse el ruido.
- El riego, los programas y la lógica de lluvia **no dependen de LVGL** y siguen funcionando con la pantalla pausada.

Si lo ves necesario puedes comprobarlo alumbrando con una linterna la pantalla apagada: se aprecia el ruido en el panel.

## Lógica de lluvia

### Cómo funciona

1. Mientras el sensor marca lluvia, el controlador cuenta los segundos en una **ventana deslizante de 24 horas** (un contador por hora). Se guarda en memoria no volátil, así que sobrevive a reinicios y cortes de luz. Se usa una ventana deslizante y no el día natural para que la lluvia de la noche cuente para el riego de primera hora de la mañana.
2. Cuando llega la hora de inicio de un **programa automático**, decide:
   - Si **está lloviendo en ese momento**, no arranca.
   - Si los minutos de lluvia de las últimas 24 h son **≥ "Saltar desde"**, se salta ese programa.
   - Si son **≥ "Reducir desde"**, riega con el tiempo reducido (por defecto, el 50 %). Se multiplica por el multiplicador de tiempo, sin sobrescribirlo.
   - En otro caso, riega normal.
3. Al saltar un programa, el siguiente programa vuelve a evaluar. Como la ventana se desliza, el riego se recupera solo cuando la lluvia "envejece".
4. **El riego manual no se ve afectado nunca.**

Los tiempos de cada zona se **redondean al minuto**: una zona de 1 minuto con un factor bajo puede quedarse en 0 minutos y no regar.

### Página Lluvia

| Control | Para qué sirve | Rango | Por defecto |
|---|---|---|---|
| **Sensor: ACTIVO / DESACTIVADO** | Activa o desactiva toda la lógica de lluvia | – | Activo |
| **Reducir desde (min)** | Minutos de lluvia en 24 h a partir de los cuales se reduce el riego | 0–240 | 10 |
| **Riego reducido (%)** | Porcentaje de tiempo cuando se reduce | 0–100 | 50 |
| **Saltar desde (min)** | Minutos de lluvia en 24 h a partir de los cuales se salta el programa | 0–720 | 45 |
| **Borrar (mantener)** | Borra el histórico de lluvia de las últimas 24 h (mantén pulsado para evitar borrarlo sin querer) | – | – |

Los valores suben y bajan de 5 en 5. Arriba se muestran los minutos de lluvia de las últimas 24 h y abajo la última decisión tomada.

Un umbral a **0** desactiva ese tramo.

**Con el sensor DESACTIVADO**, el sensor de lluvia se ignora por completo: no se cuenta nada, no impide arrancar los riegos y el riego automático sale siempre al 100 %. Es útil si el sensor falla o se queda enganchado. El histórico no se acumula mientras está desactivado.

### Cómo calibrarlo

El sensor mide **tiempo**, no milímetros, y puede tardar en secarse tras la lluvia. Los valores por defecto son un punto de partida:

1. Observa unas semanas los minutos de lluvia de 24 h en la página Lluvia.
2. Ajusta los umbrales a tu jardín y al clima.

Para probar sin esperar a la lluvia, baja los umbrales a unos 5 y 10 minutos y simula la lluvia cerrando el contacto del sensor (GPIO15 a GND con el cableado por defecto).

## Entidades en Home Assistant (opcional)

Además de las entidades de zonas, programas y ajustes, la lógica de lluvia expone:

| Entidad | Tipo |
|---|---|
| Sensor de lluvia activo | switch |
| Lluvia últimas 24 h | sensor (min) |
| Última decisión de riego | sensor de texto |
| Borrar histórico de lluvia | botón |
| Lluvia: reducir riego desde / tiempo de riego reducido / saltar riego desde | números |

## Solución de problemas

**Pantalla negra y reinicios continuos (`CORRUPT HEAP`, `Interrupt wdt timeout` en el log).**
Es un bucle infinito entre los rodillos de hora de los programas y sus selects, que con versiones recientes de ESPHome/LVGL desborda la pila al arrancar. Este YAML ya lo evita: en cada rodillo, el `on_value` solo cambia el select si la hora es distinta (busca `evita el bucle`). Si lo has modificado, conserva esa comprobación.

**`xl9535 was marked as failed` en el log.**
La placa de relés no responde por I2C. Comprueba su alimentación (al flashear solo por USB puede estar sin tensión), el cableado de SDA/SCL y la dirección `0x20`.

**Al despertar la pantalla se ve ruido o la interfaz no se redibuja.**
Es la protección anti burn-in: al tocar, el controlador reanuda LVGL (`lvgl.resume` y `lvgl.widget.redraw`) y espera 200 ms antes de encender la luz. Si lo has modificado, conserva ese orden. Si tu panel lo necesita, sube ese retardo.

**La placa se queda en modo seguro (`Boot loop detected`).**
Tras 10 arranques fallidos ESPHome arranca solo con WiFi y OTA. Flashea por cable un firmware que arranque bien.

**La pantalla no inicia o sale con los colores invertidos tras conectar el reloj.**
Una causa probable es el reloj alimentado a 5 V. Muchos módulos (por ejemplo, los típicos con DS1307) llevan las resistencias pull-up de SDA/SCL conectadas a su alimentación, y a 5 V ponen el bus I2C compartido por encima de los 3,3 V del ESP32. En un montaje así la pantalla no llegó a iniciarse. Alimenta el módulo a **3,3 V** y comprueba que SDA y SCL en reposo están a unos 3,3 V.

**No riega de forma automática y la hora de arriba es incorrecta.**
Sin hora válida (sin WiFi y sin reloj) el programador no arranca ningún programa. Conecta el WiFi (vale el del móvil) y pulsa Ajustes > Reloj > *Sincronizar ahora*, o conecta un reloj PCF8563 para que conserve la hora. El riego manual no depende de la hora.

**Error de glifos o iconos que no salen.**
Falta `fonts/materialdesignicons-webfont.ttf` o es de una versión muy antigua.

**No se actualiza por WiFi (OTA).**
El OTA usa la misma clave que la API (`api_encryption_key`). Comprueba que el ESPHome desde el que subes es reciente y que usas la misma clave que está grabada en la placa.

## Estructura del repositorio

```
riego-esp-lvgl.yaml      configuración completa
secrets.yaml.example     plantilla de claves (copia a secrets.yaml)
fonts/                   iconos Material Design Icons (ver fonts/README.md)
README.md                este manual
docs/img/                capturas de la pantalla usadas en este manual
LICENSE
```

## Licencia

Este proyecto es **código abierto** y se publica bajo la licencia **MIT** (ver `LICENSE`).

Existe gracias al software libre sobre el que se apoya:

- **[ESPHome](https://esphome.io)**: licencia mixta. Su código Python está bajo MIT y el código C++ del firmware (runtime) bajo **GPLv3**.
- **[LVGL](https://lvgl.io)**: MIT.
- **[Material Design Icons](https://pictogrammers.com/library/mdi/)** (Pictogrammers): licencia Apache 2.0. Roboto la descarga ESPHome desde Google Fonts y tiene su propia licencia abierta.

Compartir el fichero YAML de configuración, como hace este repositorio, es una cosa. **Distribuir firmware ya compilado** es otra, porque incluye el código GPLv3 de ESPHome y obliga a cumplir esa licencia (entre otras cosas, ofrecer el código fuente). Esto es solo una aclaración general y no asesoramiento legal.

## Créditos

Hecho con ESPHome y LVGL. Gracias a sus comunidades por hacerlo posible.
