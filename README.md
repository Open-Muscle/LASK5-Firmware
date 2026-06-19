# LASK5 Firmware

MicroPython firmware for the OpenMuscle LASK5 hand labeler. ESP32-S3 host, 4 piston-pressure sensors, joystick, OLED, and dual transport: Wi-Fi UDP for hub-coordinated streaming and ESP-NOW for low-latency P2P to an OpenHand robot.

> **Status: v3.0.0 (June 2026).** Implements the OpenMuscle device-discovery + subscribe protocol. The Open Muscle Connect Android app and the PC's Open Muscle Lab can both discover this device on the local Wi-Fi and subscribe to its label frame stream. The ESP-NOW path to OpenHand is preserved byte-for-byte from earlier firmware.

## What this firmware does

- Reads 4 piston pressure sensors and the joystick at the configured rate (default 25 Hz)
- Announces itself on the local Wi-Fi network via mDNS hostname and a UDP broadcast beacon. No hardcoded destination IP.
- Accepts hub subscriptions via a TCP command channel; unicasts each label frame to every subscribed hub
- Drives an OLED with a live taskbar and an on-device menu for calibration and mode switching
- Continues to broadcast scaled label values to a paired OpenHand robot over ESP-NOW (unchanged from earlier firmware)

## Repo layout

```
LASK5-Firmware/
├── README.md
├── LICENSE                       (MIT, OpenMuscle)
├── boot.py                       (boot entry, instantiates LASK5)
├── labeler.py                    (main application class + async tasks)
├── sensor_pistons.py             (4 piston ADC reads + calibration)
├── display_lask5.py              (OLED taskbar + menu rendering)
├── splash_frames.py              (boot splash animation frames)
├── config/
│   └── defaults.json             (default settings; settings.json gets created on first boot)
└── lib/                          (shared OpenMuscle library, vendored)
    ├── om_logger.py
    ├── om_settings.py            (NVS-backed settings with legacy-key migration)
    ├── om_device.py              (BaseDevice lifecycle)
    ├── om_network.py             (Wi-Fi + UDP + ESP-NOW + send_udp_to_subscribers)
    ├── om_display.py             (SSD1306 wrapper)
    ├── om_packet.py              (v1.0 envelope builder)
    ├── om_subscribers.py         (NEW: subscriber list with heartbeat timeout)
    ├── om_discovery.py           (NEW: mDNS hostname + UDP broadcast announce)
    └── om_commands.py            (NEW: TCP command server + standard handlers factory)
```

## Protocol summary

Implements the OpenMuscle v1.0 protocol per the canonical specs:
- Discovery + transport spec: [`OpenMuscle-Connect/docs/DEVICE-DISCOVERY-SPEC.md`](https://github.com/Open-Muscle/OpenMuscle-Connect/blob/main/docs/DEVICE-DISCOVERY-SPEC.md)
- Wire format: [`OpenMuscle-Connect/docs/WIRE-FORMAT.md`](https://github.com/Open-Muscle/OpenMuscle-Connect/blob/main/docs/WIRE-FORMAT.md)

Announce payload (mDNS hostname + UDP broadcast on port 3141):
```json
{
  "v": "1.0",
  "type": "announce",
  "id": "lask5-01",
  "role": "source",
  "dev": "lask5",
  "fw": "v3.0.0",
  "transports": ["wifi"],
  "caps": ["label", "status", "cmd"],
  "services": { "label": 3141, "cmd": 8002 },
  "pistons": 4,
  "joystick": true
}
```

Label frame payload (UDP to subscribers at the configured sample rate):
```json
{
  "v": "1.0",
  "type": "lask5",
  "id": "lask5-01",
  "ts": 61588,
  "data": {
    "values": [1.0, 1.0, 0.99, 1.0],
    "joystick": {"x": 1945, "y": 1937}
  }
}
```

`values` are calibrated to 0.0..1.0 floats per piston. `joystick.x` and `joystick.y` are raw ADC readings (0..4095).

## ESP-NOW broadcast (unchanged)

The labeler still supports stream_mode = "espnow" for direct broadcast to a paired bracelet or OpenHand robot, bypassing Wi-Fi entirely. Wire format matches the earlier monolithic firmware: a string repr of `[p1, p2, p3, p4, jx]` scaled to 0..800. Existing receivers continue to work without changes.

## How to flash

Files live at the device root and under `/lib/`. Use any MicroPython upload tool that can write to the on-flash filesystem. `mpremote` is the standard option:

```bash
cd LASK5-Firmware
mpremote connect COM<n> cp boot.py :boot.py
mpremote connect COM<n> cp labeler.py :labeler.py
mpremote connect COM<n> cp sensor_pistons.py :sensor_pistons.py
mpremote connect COM<n> cp display_lask5.py :display_lask5.py
mpremote connect COM<n> cp splash_frames.py :splash_frames.py
mpremote connect COM<n> cp lib/*.py :lib/
mpremote connect COM<n> exec "import machine; machine.reset()"
```

If mpremote fails to enter raw REPL because the device's logger floods the UART, use a direct pyserial uploader. One example lives in this repo's history as `_upload_lask5.py`.

## Settings

First boot creates `config/settings.json` from `config/defaults.json`. After that, settings can be edited in place over USB serial or via the on-device menu where supported. Key fields:

- `device_id`: the announce identifier ("lask5-01" by default)
- `wifi_ssid` / `wifi_password`: network to join
- `sample_rate_hz`: label frame rate (default 25)
- `stream_mode`: "udp" or "espnow"
- `cmd_port`: TCP command channel (default 8002, chosen to avoid collision with FlexGrid's 8001)

## License

MIT. See `LICENSE`.
