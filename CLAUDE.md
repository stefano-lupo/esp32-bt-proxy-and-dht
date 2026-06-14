# esp32-bt-proxy-and-dht — Agent context

Two ESP32 devices maintaining persistent BLE connections to Tuiss SmartView motorised blinds, exposed as `cover` entities in Home Assistant.

## Devices

| Device | YAML | IP | MAC | Blind model |
|--------|------|----|-----|-------------|
| living-room-esp32 | `living-room-esp32.yaml` | `192.168.0.118` | `EC:A5:21:CA:9A:F8` | TS2900 |
| bedroom-esp32 | `bedroom-esp32.yaml` | `192.168.0.142` | `FD:74:46:D2:50:0F` | TS5001 |

## Critical model differences

**TS2900 (living room):** Only needs CONNECTION_MESSAGE (`ff 03 03 03 03 78 78 78 78 78 78`) once on first boot. Handled by `lr_initialized` global.

**TS5001 (bedroom):** Requires CONNECTION_MESSAGE **+ timestamp** on **every** connection. Without the timestamp it severs the BLE connection after ~5 seconds (`HCI_ERR_REMOTE_USER_TERM_CONN`, reason `0x13`). The timestamp is sent via SNTP time lambda on every `on_connect`. The `br_initialized` guard that the living room uses must NOT be present in the bedroom config.

## Known ESPHome 2026.3 gotcha

**`wait_until` silently hangs inside cover template actions.** Do not use it there. Use a direct `if(br_ready)` check instead.

## Flashing

```bash
# Copy then flash
cp bedroom-esp32.yaml /home/server/docker/esphome/config/
docker exec esphome esphome run /config/bedroom-esp32.yaml --device 192.168.0.142

cp living-room-esp32.yaml /home/server/docker/esphome/config/
docker exec esphome esphome run /config/living-room-esp32.yaml --device 192.168.0.118

# If OTA fails with "Connection reset": restart ESPHome first
cd /home/server/docker && docker compose restart esphome

# If new firmware isn't taking: clear build cache
docker exec esphome rm -rf /config/.esphome/build/bedroom-esp32
```

## Debugging

**Seq** (http://192.168.0.99:5341):
- `Application = 'bedroom-esp32'` — bedroom logs
- Filter `dropped` → commands that arrived while BLE wasn't ready
- Rapid connect/disconnect every 5s = TS5001 handshake failure (timestamp not sent or SNTP not synced)

**HA entities:**
- `binary_sensor.bedroom_esp_32_bt_proxy_dht11_bedroom_blind_connected`
- `binary_sensor.living_room_esp32_bt_proxy_dht_living_room_blind_connected`

## Session logs

Full history of debugging attempts and outcomes in `sessions/YYYY-MM-DD.md`. Read the most recent one before making changes.

## Do not commit

`secrets.yaml` is gitignored. Never commit it. Device YAMLs use only `!secret` references.
