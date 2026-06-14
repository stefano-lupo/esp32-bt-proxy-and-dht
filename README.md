# esp32-bt-proxy-and-dht

ESPHome firmware for two ESP32 devices, each maintaining a persistent BLE connection to a Tuiss SmartView motorised blind and exposing it as a `cover` entity in Home Assistant. Also includes a DHT11 temperature/humidity sensor on each device.

## Devices

| Device | Config | IP | MAC (blind) | Blind model |
|--------|--------|----|-------------|-------------|
| `living-room-esp32` | `living-room-esp32.yaml` | `192.168.0.118` | `EC:A5:21:CA:9A:F8` | TS2900 |
| `bedroom-esp32` | `bedroom-esp32.yaml` | `192.168.0.142` | `FD:74:46:D2:50:0F` | TS5001 |

## Flashing

Edit the relevant yaml, copy to the ESPHome config directory, then flash OTA:

```bash
# Living room
cp living-room-esp32.yaml /home/server/docker/esphome/config/
docker exec esphome esphome run /config/living-room-esp32.yaml --device 192.168.0.118

# Bedroom
cp bedroom-esp32.yaml /home/server/docker/esphome/config/
docker exec esphome esphome run /config/bedroom-esp32.yaml --device 192.168.0.142
```

If the flash fails with `Connection reset by peer`, restart ESPHome first:
```bash
cd /home/server/docker && docker compose restart esphome
```

`secrets.yaml` is gitignored — the live copy lives at `/home/server/docker/esphome/config/secrets.yaml`.

## BLE Protocol (Tuiss SmartView)

Both blinds use the same service/characteristic UUIDs and command bytes, sourced from [pink88/Tuiss2HA](https://github.com/pink88/Tuiss2HA).

**Service UUID:** `00010203-0405-0607-0809-0a0b0c0d1910`  
**Write characteristic:** `00010405-0405-0607-0809-0a0b0c0d1910`  
**Notify characteristic:** `00010304-0405-0607-0809-0a0b0c0d1910`

| Command | Bytes |
|---------|-------|
| Connection message | `ff 03 03 03 03 78 78 78 78 78 78` |
| Timestamp | `ff 78 ea 41 02 00 YY MM DD HH mm SS` (YY = year−2000) |
| Open | `ff 78 ea 41 bf 03 e8 03` |
| Close | `ff 78 ea 41 bf 03 00 00` |
| Stop | `ff 78 ea 41 5f 03 01` |
| Set position | `ff 78 ea 41 bf 03 <pos_byte> <group_byte>` |

Position encoding (0% = closed, 100% = open):
```
pct       = (1 - pos) * 100
pos_byte  = ((100 - pct) * 10) % 256
group_byte: pct > 74 → 0x00, > 48 → 0x01, > 23 → 0x02, else → 0x03
```

## Why the two configs differ

The TS2900 (living room) and TS5001 (bedroom) behave differently over BLE:

**TS2900** — tolerates a persistent connection with only the connection message sent once on first boot. Cover actions write directly without checking connection state.

**TS5001** — requires the full handshake (connection message + timestamp) on *every* connection. Without the timestamp it terminates the connection after ~5 seconds (`HCI_ERR_REMOTE_USER_TERM_CONN`, reason `0x13`). The handshake is therefore sent on every `on_connect` event, with `br_ready` set only after both writes complete. Cover actions guard on `br_ready` and log a warning if a command arrives during the brief reconnect window.

**ESPHome note:** `wait_until` silently hangs inside cover template actions in ESPHome 2026.3 — do not use it there.

## HA Entities

Each device exposes:

| Entity | Type | Notes |
|--------|------|-------|
| `Living Room Blind` / `Bedroom Blind` | `cover` | open/close/stop/position |
| `Living Room Blind Connected` / `Bedroom Blind Connected` | `binary_sensor` | BLE connection state |
| `DHT22 Temperature` | `sensor` | |
| `DHT22 Humidity` | `sensor` | |
| `Restart` | `button` | OTA-safe restart |

## Monitoring

The `Bedroom Blind Connected` binary sensor can be used in a HA automation to alert if the BLE connection drops. Failed commands (blind not connected when HA sends one) log at `WARN` level and are visible in Seq under `Application = 'bedroom-esp32'` filtered for `dropped`.
