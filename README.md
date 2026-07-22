# Fantech Hero HRV — ESPHome Controller

ESPHome firmware that replaces the Fantech ECO-Touch wall controller on a
**Fantech Hero HRV** unit. The ESP32 acts as the Modbus RTU master on the
RS-485 bus, exposing full control to Home Assistant via the ESPHome native
API while also providing a local display and physical button interface. Power
is drawn via 12v supply from the fan unit, making this an easy replacement with
no additional wiring.

## Attribution

The Modbus protocol for the Fantech Hero was reverse-engineered by **Dustin Thomas**:
This code is derived from his work:

> https://dustint.com/post/2024-07-30-fantech-hrv/

This ESPHome implementation was developed with [Claude](https://claude.ai) (Anthropic).

## Hardware

As built, this firmware runs on an M5 Stack Station with RS485:

> https://shop.m5stack.com/products/m5stack-station-esp32-iot-development-kit-rs485-version?variant=43165398827265

- Draws power from the vent unit via PWR485 on existing cable.
- Three physical buttons on GPIO37 / GPIO38 / GPIO39 for manual setting low/medium/high
- AXP192 power management IC (I2C, address 0x34)
- ST7789V 135×240 LCD (SPI) shows current status
- AXP192 PEK button used as power/on-off toggle
- RS-485 transceiver connected to UART0 (GPIO1 TX / GPIO3 RX), flow control on GPIO2
- esphome sensors for control from Home Assistant

## Modbus Protocol

| Register | Type   | Description                     |
|----------|--------|---------------------------------|
| 0x3E     | U_WORD | Mode (see values below)         |
| 0x34     | U_WORD | High/Timed Max timer (seconds)  |
| 0x12     | U_WORD | Intake fan speed (balancing)    |
| 0x13     | U_WORD | Exhaust fan speed (balancing)   |
| 0x2F     | S_WORD | Intake air temperature (°C)     |
| 0x28     | U_WORD | Current actual operating mode   |

**Mode register values:** Off=2, Vent Low=3, Vent Med=4, Vent High=5,
Timed Max=6, Recirc Low=7, Recirc Med=8, Recirc High=9

Fan speed registers use a two's complement-style encoding:
`0` = 100%, otherwise `value = 65536 - (100 - percent)`.

## Features

- **Mode select** — all eight HRV modes exposed to Home Assistant
- **Timed Max** — HA service call `timed_max(time_minutes)` activates high
  ventilation for a set duration, then restores the previous mode
- **Fan balancing** — intake/exhaust speed trim exposed as config entities
- **Comms watchdog** — display shows "No Comms" if Modbus goes silent >90s
- **Physical buttons** — A/B/C set Low/Med/High; Power button toggles on/off
- **Power & energy** — estimated from mode (measured at 60/100/132W)
- **Diagnostic registers** — several unknown registers polled for investigation

## Setup

1. Add to your ESPHome `secrets.yaml`:

   ```yaml
   wifi_ssid: "YourSSID"
   wifi_password: "YourPassword"
   ota_password: "YourOTAPassword"
   ```

2. Flash with ESPHome:

   ```bash
   esphome run fantech-hrv-bedrooms.yaml
   ```

3. Add the device to Home Assistant via the ESPHome integration.

## Wiring

```

Disconnect the Eco-touch and move the wires to the bottom port on the M5Stack Station:

D+ -> (red)V  12v
down-arrow -> (black)G  ground
A -> A
B -> B (swap these two if no comms)
```

## Known Unknowns

Several diagnostic registers (0x04, 0x1F, 0x20, 0x39, 0x5F, 0x60, 0x62)
are polled but their meaning is not yet determined. Contributions welcome.
