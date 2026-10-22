# Temperature & Humidity Monitor (STM32 + DHT11 + LCD1602)

A small embedded project that reads temperature and humidity from a DHT11 sensor using an STM32 Nucleo-G491RE and shows the live values on a 16x2 LCD1602 display driven in 4-bit parallel mode.

<!-- Add a photo of the finished setup here. Put the image in an /images folder, e.g.: -->
<!-- ![Project photo](images/setup.jpg) -->

## Features

- Reads temperature and humidity from a DHT11 over its single-wire protocol, using a hardware timer (TIM6) as a microsecond counter to measure pulse widths
- Displays live readings on a 16x2 LCD1602 (HD44780-compatible)
- LCD driven in 4-bit parallel mode over GPIO
- Written in C with the STM32 HAL library (GPIO and timer)
- Custom DHT11 and HD44780 drivers written from scratch (no external libraries)
- Checksum validation of the 40-bit DHT11 frame
- Timeout on the sensor's start response, with a "DHT ERROR" message on the display if the sensor does not answer

## Hardware

| Component | Details |
|-----------|---------|
| Microcontroller board | STM32 Nucleo-G491RE (ARM Cortex-M4 with FPU) |
| Sensor | DHT11 temperature and humidity sensor |
| Display | LCD1602 (16x2 characters, HD44780 controller) |
| Other | Breadboard, jumper wires, [10k potentiometer for contrast, resistors if used] |

## Wiring

### DHT11

| DHT11 pin | Nucleo connection |
|-----------|-------------------|
| VCC | 3.3 V |
| DATA | [GPIO pin, e.g. PA0] |
| GND | GND |

> Some DHT11 modules include a pull-up resistor on the data line. If you use the bare sensor, add a 4.7k-10k pull-up between DATA and VCC.

### LCD1602 (4-bit mode)

| LCD pin | Name | Nucleo connection |
|---------|------|-------------------|
| 1 | VSS | GND |
| 2 | VDD | 5V |
| 3 | V0 | Contrast (resistors) |
| 4 | RS | [GPIO pin] |
| 5 | RW | GND (write-only) |
| 6 | E | [GPIO pin] |
| 11 | D4 | [GPIO pin] |
| 12 | D5 | [GPIO pin] |
| 13 | D6 | [GPIO pin] |
| 14 | D7 | [GPIO pin] |
| 15 | A | Backlight anode (through resistor if needed) |
| 16 | K | GND |

> Pins D0-D3 are left unconnected in 4-bit mode.

## How it works

### DHT11 reading

The DHT11 uses a proprietary single-wire protocol. The microcontroller sends a start signal, the sensor replies with a response pulse, and then sends 40 bits (humidity integer and decimal, temperature integer and decimal, and an 8-bit checksum). Each bit is distinguished by the length of the high pulse on the data line, so the timing matters.

The driver (`dht11.c`) handles the protocol in four steps:

1. **Start signal:** the data pin is set as an output and pulled low for 18 ms, then switched to an input with pull-up so the sensor can answer.
2. **Response check:** the sensor's response (low, then high, then low) is verified, with a timeout measured by TIM6 so the firmware does not wait forever if no sensor is connected.
3. **Bit reading:** for each of the 40 bits, the code waits for the line to go high, resets TIM6, and measures how long it stays high. TIM6 runs as a free-running microsecond counter, so a short pulse (about 26-28 µs) is read as `0` and a long pulse (about 70 µs) as `1`, using a 45 µs threshold.
4. **Checksum:** the first four bytes (humidity and temperature, integer and decimal parts) are summed and compared to the fifth byte before anything is displayed.

### LCD1602 in 4-bit mode

The HD44780 controller normally takes 8 data lines, but it can be driven with only 4 (D4-D7) by sending each byte as two nibbles. The firmware performs the HD44780 initialisation sequence to switch the controller into 4-bit mode, then sends commands and characters as high and low nibbles, pulsing the Enable pin for each transfer.

The driver (`lcd.c`) is written from scratch on top of the HAL GPIO functions. Initialisation follows the HD44780 datasheet sequence (`0x03` three times, then `0x02` to select 4-bit mode), then configures 2 lines (`0x28`), clears the display, sets the entry mode (`0x06`) and turns the display on (`0x0C`). Each byte is split into a high and low nibble, placed on D4-D7, and clocked in with a pulse on the Enable pin. RS selects between commands (low) and characters (high).

### Program flow

1. Initialise clocks, GPIO and timer
2. Initialise the LCD in 4-bit mode
3. Loop: send the start signal, verify the sensor's response, read the 40 data bits, check the checksum
4. If the checksum is valid, show temperature on line 1 and humidity on line 2 of the LCD
5. Wait at least 1 second between readings, as the DHT11 samples at 1 Hz at most [confirm the delay in your main loop]

## Getting started

### Requirements

- STM32 Nucleo-G491RE
- DHT11 sensor and LCD1602 display
- [STM32CubeIDE, edit if you used something else]
- USB cable for programming and power

## Example output

```
Temp: 23.0
Humid: 45.0
```

## DHT11 limitations

The DHT11 is a low-cost sensor with limited accuracy:

| Parameter | Range | Accuracy |
|-----------|-------|----------|
| Temperature | 0-50 °C | ±2 °C |
| Humidity | 20-80 % RH | ±5 % RH |

It also supports a maximum sampling rate of about 1 Hz. For higher accuracy, a sensor like the DHT22 or SHT31 would be a better choice.

## Project structure

```
[Adjust paths to match your repository]
├── Core/
│   ├── Inc/
│   │   ├── dht11.h
│   │   └── lcd.h
│   └── Src/
│       ├── main.c
│       ├── dht11.c
│       └── lcd.c
├── images/
└── README.md
```

## Possible improvements

- Replace the polling loops in the bit reader with edge interrupts or timer input capture, and add timeouts to every wait loop
- Stream readings over UART to a PC for logging and plotting in Python
- Send data over MQTT for remote monitoring
- Use a more accurate sensor (DHT22 or SHT31)
- Add threshold alerts (e.g. a buzzer or LED when humidity is too high)
- Use low-power modes and measure battery life

## Author

Peter Ha Zu
[LinkedIn](https://www.linkedin.com/in/peter-ha-/) | Peterhazu@gmail.com
