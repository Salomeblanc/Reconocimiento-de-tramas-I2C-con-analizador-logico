# Reconocimiento de tramas I²C con analizador lógico

Práctica de laboratorio de **Comunicaciones Digitales** · Programa de Ingeniería en Telecomunicaciones · Universidad Militar Nueva Granada · Periodo 2026-2

| | |
|---|---|
| **Autores** | Harol Felipe Riveros Sierra (1401660) · Salome Bohórquez Blanco (1401654) |
| **Docente** | Ing. José de Jesús Rugeles Uribe |
| **Guía original y código base** | <https://github.com/jrugeles/I2C> |

---

## Descripción

Se analiza el protocolo **I²C** capturando las señales SCL y SDA con un analizador lógico (Logic 2) mientras una **Raspberry Pi Pico 2W**, programada en MicroPython, se comunica con una pantalla **OLED SSD1306** (dirección `0x3C`). Con las capturas se identifican los elementos de una trama, se comparan respuestas ACK y NACK, se escanea el bus y se relacionan los bytes enviados a la pantalla con los comandos de la hoja de datos.

## Objetivos

1. Identificar START, STOP, direccionamiento de 7 bits + R/W y los estados ACK/NACK en Logic 2.
2. Verificar la respuesta de la OLED con una dirección correcta (`0x3C`) y una incorrecta (`0x3D`).
3. Medir la frecuencia de SCL con los cursores del analizador.
4. Escanear el bus con `i2c.scan()` para detectar dispositivos.
5. Reconocer los comandos y datos enviados a la OLED (byte de control, comando/dato y ACK).

## Materiales

- Raspberry Pi Pico 2W (MicroPython)
- Pantalla OLED SSD1306 128×32, I²C, dirección `0x3C`
- Analizador lógico USB de 8 canales, 24 MHz
- Software Logic 2 (con el decodificador I²C habilitado)
- Protoboard y cables de conexión

## Conexiones

| Señal | Pico 2W | Analizador lógico | OLED |
|---|---|---|---|
| SCL (I²C1) | GP15 | CH0 | SCL |
| SDA (I²C1) | GP14 | CH1 | SDA |
| GND | GND | GND (común) | GND |
| Alimentación | 3V3 | (no se conecta) | VCC |

> Verificar en el módulo OLED que la alimentación sea compatible con 3.3 V.

**Frecuencia de muestreo:** configurar el analizador a ≥ 10 × f<sub>SCL</sub> (por ejemplo, ≥ 1 MS/s para un bus de 100 kHz).

## Estructura sugerida del repositorio

```
.
├── README.md
├── codigo/
│   ├── OLED_ADDR_test.py      # Prueba ACK/NACK (0x3C / 0x3D)
│   ├── scan_i2c_addr.py       # Escaneo de direcciones con i2c.scan()
│   └── OLED_demo_menu.py      # Menú por consola para la OLED
├── informe/
│   ├── 8 INFORME COMUNICACION DIGITAL (HAROL RIVEROS - SALOME BOHORQUEZ).pdf
│   └── 8 INFORME COMUNICACION DIGITAL (HAROL RIVEROS - SALOME BOHORQUEZ).docx
├── capturas/
│   └── segundo lab cd.pdf     # Capturas de Logic 2 y fotos del montaje
└── tablas/
    └── Tablas de la respectiva práctica.xlsx
```

## Cómo reproducir la práctica

1. Cargar MicroPython en la Pico 2W y conectar todo según la tabla de conexiones.
2. En Logic 2, asignar CH0 → SCL y CH1 → SDA, agregar el analizador **I²C** y fijar la frecuencia de muestreo.

### Parte 1 · ACK y NACK

1. Ejecutar `codigo/OLED_ADDR_test.py` con `ADDR = 0x3C`. Iniciar la captura antes de pulsar *Run* (el script espera 1 s).
2. Identificar **START → 0x78 → ACK → STOP** y medir f<sub>SCL</sub> con los cursores.
3. Cambiar a `ADDR = 0x3D`, repetir y comprobar que en el bit 9 SDA queda en alto (**NACK**).

### Parte 2 · Escaneo del bus

1. Iniciar la captura y ejecutar `codigo/scan_i2c_addr.py`.
2. En consola debe aparecer `0x3c`; en Logic 2, sondeos con NAK y un solo ACK en `0x3C`.

### Parte 3 · Comandos de la OLED

1. Ejecutar `codigo/OLED_demo_menu.py`. El programa escanea el bus, detecta la OLED y muestra un menú (bus a 50 kHz):
   `1` Apagar (`0xAE`) · `2` Encender (`0xAF`) · `3` Contraste (0-255) · `4` Invertir 1/0 · `5` Limpiar · `6` Texto demo · `7` Animación breve · `8` Comando RAW · `9` Dato RAW · `F` Cambiar frecuencia I²C · `0` Salir
2. Capturar cada opción y comparar los bytes con la hoja de datos.

## Resultados principales

### ACK vs NACK

| Elemento | Captura ACK | Captura NACK |
|---|---|---|
| Start detectado (SDA↓ con SCL alto) | Sí | Sí |
| Octeto / etiqueta del decodificador | `0x78` / Write to [0x3C] + ACK | `0x7A` / Write to [0x3D] + NAK |
| Bit 9 (ACK = 0 / NACK = 1) | ACK (SDA = 0) | NACK (SDA = 1) |
| Stop detectado (SDA↑ con SCL alto) | Sí | Sí |
| Frecuencia medida de SCL | 83.07 kHz | 83.43 kHz |

El octeto se obtiene como `(ADDR << 1) | R/W`: `0x3C → 0x78` y `0x3D → 0x7A`.

### Mediciones con cursores

| Medición | Tiempo | Frecuencia equivalente |
|---|---|---|
| Octeto `0x78` (8 bits) | 84.103 µs | ≈ 95 kHz (10.5 µs/bit) |
| Bit 9, captura ACK | 13.004 µs | 76.9 kHz |
| Bit 9, captura NACK | 12.986 µs | 77.01 kHz |
| Período de SCL, captura ACK | 12.038 µs | 83.07 kHz |
| Período de SCL, captura NACK | 11.987 µs | 83.43 kHz |
| Período de SCL en el menú OLED (configurado a 50 kHz) | 21 µs | ≈ 47.6 kHz |

Con 100 kHz configurados se midieron ≈ 83 kHz, por lo que conviene verificar la frecuencia real del bus con el analizador.

### Escaneo

`i2c.scan()` devolvió `['0x3c']`. En la traza, los sondeos a otras direcciones (por ejemplo `0x3B` y `0x3D`) terminan en NAK y solo `0x3C` responde con ACK.

### Comandos observados en la OLED

| Opción | Bytes observados tras la dirección | Función (hoja de datos SSD1306) |
|---|---|---|
| Apagar | `0x80, 0xAE` | Set Display OFF |
| Encender | `0x80, 0xAF` | Set Display ON |
| Contraste (128) | `0x80, 0x81` y luego `0x80, 0x80` | Set Contrast Control (comando + valor) |
| Invertir = 1 | `0x80, 0xA7` | Inverse Display |
| Invertir = 0 | `0x80, 0xA6` | Normal Display |
| Texto/animación | `0x80, 0x03` y luego `0x40, 0x00, 0x00, 0x41, 0x7F, …` | Comando de columna y escritura de datos en la GDDRAM |

El byte de control `0x80` precede a los comandos (Co = 1, D/C# = 0) y `0x40` a los datos de pantalla (Co = 0, D/C# = 1).

## Conclusiones

- Una transacción de escritura I²C se compone de START, octeto de dirección + R/W, bit de reconocimiento y STOP.
- Entre `0x3C` y `0x3D` solo cambian el octeto (`0x78` / `0x7A`) y el nivel de SDA en el bit 9 (ACK = 0, NACK = 1).
- La frecuencia real de SCL puede diferir de la configurada; se midieron ≈ 83 kHz (100 kHz configurados) y ≈ 47.6 kHz (50 kHz configurados).
- `i2c.scan()` detectó únicamente la OLED en `0x3C`.
- Los bytes capturados coinciden con la hoja de datos del SSD1306 y con el efecto visible en la pantalla.

## Referencias

1. NXP Semiconductors, *UM10204: I²C-bus specification and user manual*, Rev. 7.0, 2021.
2. Solomon Systech, *SSD1306: 128 × 64 Dot Matrix OLED/PLED Segment/Common Driver with Controller*, Datasheet.
3. Raspberry Pi Ltd, *Raspberry Pi Pico 2 W*, Datasheet. <https://www.raspberrypi.com/documentation/microcontrollers/>
4. MicroPython, *class I2C*, documentación de `machine`. <https://docs.micropython.org/en/latest/library/machine.I2C.html>
5. Saleae, *Logic 2 Software*. <https://www.saleae.com/>
6. J. Rugeles, *Reconocimiento de tramas I2C con analizador lógico*, guía de laboratorio. <https://github.com/jrugeles/I2C>
