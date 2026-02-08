# mbed_SC16IS750

A fork of the SC16IS750/SC16IS752 UART bridge library adapted for **mbed-based Arduino cores** (including **Arduino GIGA R1**), with an API shaped around `mbed::Stream`, `mbed::SPI`, and `mbed::I2C` types.

This library lets your MCU talk to NXP SC16IS75x devices over SPI or I2C, and exposes a serial-like interface (`getc`, `putc`, `readable`, `writable`, formatting, baud, flow control, FIFO/GPIO helpers).

---

## Why this fork

This fork is intended to avoid integration friction in environments where Arduino and mbed symbols can clash.

Key choices in this codebase:

- Uses mbed-native classes (`mbed::Stream`, `mbed::SPI`, `mbed::I2C`, `mbed::DigitalOut`) directly.
- Public API mirrors classic serial behavior but stays mbed-centric.
- Supports both single-UART SC16IS750 and dual-UART SC16IS752 variants.
- Includes both SPI and I2C transport implementations.

---

## Supported chips and transports

### Chips

- **SC16IS750** (single UART)
- **SC16IS752** (dual UART, channel A/B selectable)

### Transports

- **SPI**
- **I2C**

---

## Features

- Serial-like operations:
  - `int getc()` (non-blocking read, returns `-1` if empty)
  - `int putc(int c)` (blocking write)
  - `readable()`, `readableCount()`, `writable()`, `writableCount()`
  - `writeString(const char*)`, `writeBytes(const char*, int)`
- UART configuration:
  - `baud(...)`
  - `format(bits, parity, stop_bits)`
  - `send_break()`, `set_break(...)`
- Hardware flow control support:
  - `set_flow_control(...)`
  - `set_flow_triggers(resume, halt)`
- FIFO configuration/reset helpers:
  - `set_fifo_control()`
  - `flush()`
- SC16IS75x GPIO helpers:
  - `ioSetDirection(...)`, `ioSetState(...)`, `ioGetState()`
- Reset/control helpers:
  - `swReset()`
  - optional reset pin + `hwReset()`
- Connectivity check:
  - `connected()` scratch-register read/write test

---

## Installation

### Arduino IDE (ZIP)

1. Download this repository as ZIP.
2. In Arduino IDE: **Sketch → Include Library → Add .ZIP Library...**
3. Include:

```cpp
#include <SC16IS750.h>
```

### PlatformIO

Add to `platformio.ini`:

```ini
lib_deps =
  https://github.com/h3cx/mbed_SC16IS750.git
```

Then include in code:

```cpp
#include <SC16IS750.h>
```

---

## Addressing note (I2C)

The library address constants (e.g. `SC16IS750_SA0`) are **8-bit shifted addresses** as traditionally used by mbed `I2C::write/read` APIs.

If you pass a custom address, use the same convention expected by the class constructor (the implementation stores `deviceAddress & 0xFE`).

---

## Quick start

> The examples below are generic mbed-style examples. Adjust pin names for your board variant and wiring.

### 1) SC16IS750 over SPI

```cpp
#include <Arduino.h>
#include <SC16IS750.h>

// Replace with your board pin names
mbed::SPI spi(MOSI, MISO, SCK);
SC16IS750_SPI uart_bridge(&spi, D10 /* CS */, NC /* optional reset */);

void setup() {
  Serial.begin(115200);

  uart_bridge.baud(115200);
  uart_bridge.format(8, ParityNone, 1);

  if (!uart_bridge.connected()) {
    Serial.println("SC16IS750 not detected");
    while (1) {}
  }

  uart_bridge.writeString("Hello from GIGA via SC16IS750!\r\n");
}

void loop() {
  // Bridge SC16IS750 RX -> USB serial monitor
  while (uart_bridge.readable()) {
    int c = uart_bridge.getc();
    if (c >= 0) {
      Serial.write((char)c);
    }
  }

  // Bridge USB serial monitor -> SC16IS750 TX
  while (Serial.available()) {
    uart_bridge.putc(Serial.read());
  }
}
```

### 2) SC16IS750 over I2C

```cpp
#include <Arduino.h>
#include <SC16IS750.h>

// Replace SDA/SCL with your board pin names if needed
mbed::I2C i2c(SDA, SCL);
SC16IS750_I2C uart_bridge(&i2c, SC16IS750_DEFAULT_ADDR, NC /* optional reset */);

void setup() {
  Serial.begin(115200);

  uart_bridge.baud(9600);
  uart_bridge.format(8, ParityNone, 1);

  if (!uart_bridge.connected()) {
    Serial.println("SC16IS750 not detected");
    while (1) {}
  }

  uart_bridge.writeString("Hello over I2C\r\n");
}

void loop() {
  while (uart_bridge.readable()) {
    int c = uart_bridge.getc();
    if (c >= 0) Serial.write((char)c);
  }
}
```

### 3) SC16IS752 channel selection

For the dual-UART SC16IS752, pick a channel at construction:

```cpp
SC16IS752_SPI uartA(&spi, D10, NC, SC16IS750::Channel_A);
SC16IS752_SPI uartB(&spi, D9,  NC, SC16IS750::Channel_B);
```

I2C variant:

```cpp
SC16IS752_I2C uartA(&i2c, SC16IS750_DEFAULT_ADDR, NC, SC16IS750::Channel_A);
SC16IS752_I2C uartB(&i2c, SC16IS750_DEFAULT_ADDR, NC, SC16IS750::Channel_B);
```

---

## Class overview

### Base class: `SC16IS750`

Abstract transport-independent serial bridge API.

Important methods:

- Serial operations: `getc`, `putc`, `readable`, `writable`, `writeString`, `writeBytes`
- UART config: `baud`, `format`, `send_break`, `set_break`
- Flow/FIFO control: `set_flow_control`, `set_flow_triggers`, `set_fifo_control`, `flush`
- Device helpers: `connected`, `ioSetDirection`, `ioSetState`, `ioGetState`, `swReset`

### Concrete classes

- `SC16IS750_SPI`
- `SC16IS750_I2C`
- `SC16IS752_SPI`
- `SC16IS752_I2C`

All concrete constructors call internal initialization (`_init()`), so normal register setup happens automatically.

---

## Wiring tips

- Keep logic levels consistent between MCU and SC16IS75x board.
- Connect common ground between MCU and peripheral boards.
- For SPI, verify mode and max clock allowed by your hardware design.
- For I2C, confirm pull-ups and actual selected address pins (A0/A1 wiring).
- Optional reset pin is recommended for robust field recovery.

---

## Known limitations / behavior notes

- XON/XOFF software flow control is defined in registers but not implemented as a high-level API workflow.
- `putc`/bulk writes are blocking when TX FIFO has no space.
- Initialization prints status via `printf(...)` in `_init()`.

---

## Troubleshooting

If `connected()` fails:

1. Re-check power and ground.
2. Confirm SPI CS wiring / I2C address selection.
3. Validate your selected constructor and pins.
4. Try slower host bus speed and shorter wiring.
5. If wired, pulse hardware reset (`hwReset()`) and reinitialize.

If serial data appears corrupted:

- Ensure both UART ends use matching baud, parity, data bits, and stop bits.
- Verify flow control expectations (RTS/CTS wiring and mode).
- Check crystal/clock assumptions on your SC16IS75x module.

---

## License

MIT. See source headers and `library.json` metadata.
