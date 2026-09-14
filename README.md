# Ikea Vindriktning

**Air quality monitor, on the original enclosure**

An [ESPHome](https://esphome.io) replacement for the stock control board inside
an IKEA VINDRIKTNING. Keeps the PM1006 particulate sensor and adds temperature,
humidity, pressure, dew point and absolute humidity on an ESP32-C3 wedged into
the same case. Reports to Home Assistant over the encrypted native API, and
optionally over MQTT.

---

## Hardware

| Amount | Item | Notes |
|--:|---|---|
| 1 | ESP32-C3-DevKitM-1 | Replaces the Vindriktning's own control board |
| 1 | PM1006 (built into the Vindriktning) | UART, 9600 baud |
| 1 | BMP280 | I²C, address `0x77` |
| 1 | AHT20 | I²C, fixed address |

```
ESP32-C3          PM1006        BMP280 / AHT20
────────          ──────        ──────────────
GPIO20  (RX) ───── TX
GPIO21  (TX) ───── RX
GPIO06  (SDA) ──────────────────── SDA
GPIO07  (SCL) ──────────────────── SCL
```

`i2c: scan: False` because both addresses are already known — `0x77` for the
BMP280 and AHT20's fixed address — so a scan at boot would only cost time
without telling anything new.

---

## Installation

Needs a `secrets.yaml` next to the config with:

```yaml
wifi_ssid: "..."
wifi_password: "..."
wifi_ap_password: "..."     # fallback AP, at least 8 characters
api_encryption_key: "..."   # base64, 32 bytes
ota_password: "..."
mqtt_broker: "..."
mqtt_username: "..."
mqtt_password: "..."
web_username: "..."         # web interface login
web_password: "..."
```

Then:

```sh
esphome run ikea-vindriktning.yaml
```

Everything after the first flash goes over the air.

---

## One config, three devices

`ikea-vindriktning.yaml` is the template: every component from `esphome:`
down. It is a complete, flashable config on its own — the living-room and
bedroom units do not duplicate any of it, they *include* it:

```yaml
substitutions:
  project_name: ikea-vindriktning-wohnen
  ...

packages:
  common: !include ikea-vindriktning.yaml
```

`packages:` merges the included file's config into the including file, with
the including file's own `substitutions:` taking priority over any of the
same name from the package — so `${project_name}`, `${friendly_name}` and
`${version}` resolve to the per-device values, and everything else (sensors,
switch, buttons, network setup) comes from `ikea-vindriktning.yaml` unchanged.

A change to a sensor, the MQTT switch, or any other shared behaviour belongs
in `ikea-vindriktning.yaml` alone; it takes effect on all three devices the
next time each is flashed.

`ikea-vindriktning-wohnen.yaml` and `ikea-vindriktning-schlafen.yaml` are
listed in `.gitignore` — only the template and its own device are tracked.

---

## Entities

### Control

| Entity | Type | Does |
|---|---|---|
| MQTT | switch | Brings the broker connection up and down without reflashing |

### Sensors

| Entity | Does |
|---|---|
| Particulate Matter 2.5µm Concentration | The Vindriktning's own PM1006 reading |
| Pressure | From the BMP280 |
| Humidity | From the AHT20 |
| Temperature | Median of the BMP280 and AHT20 readings — see below |
| Dew Point | Magnus formula, from Temperature and Humidity |
| Absolute Humidity | Grams of water per m³, from Temperature and Humidity |

`Temperature (BMP280)` and `Temperature (AHT20)` are `internal: True`: both
feed the published `Temperature` as a two-source median, so neither is shown
on its own — a single bad reading from either sensor is outvoted rather than
displayed.

### Diagnostic

`Reset Reason`, `Reset Count`, `ESPHome Version`, `Firmware Version`, `SSID`,
`BSSID`, `MAC Address`, `IP Address`, `DNS Address`, `Device Uptime`,
`WiFi Signal (dBm)`, `WiFi Signal (%)`, `Connection Status`.

`Reset Count` is a persisted counter, incremented once per boot. It answers
what `Reset Reason` cannot: that a restart happened at all while nobody was
looking. A rising count against a low `Device Uptime` is the signature of a
device rebooting in a loop. A factory reset clears it.

`Reset Reason`, `Reset Count` and `Firmware Version` are **pushed once at
boot** rather than polled. None of the three can change while the device is
up — the reason is read straight from `esp_reset_reason()`, the counter is
incremented during startup, and the version is compiled in — and
`publish_state` does not deduplicate, so polling them would resend an
identical value over the api/mqtt every `update_interval` for the lifetime of
the device.

`Reset Reason` does not go through the ESPHome `debug` component. On this
ESP32-C3 either `debug:` or `internal_temperature` stops the fallback access
point from beaconing while the rest of the firmware keeps running — found on
the sibling `infinity`/`odb2-sniffer` boards. That matters here because
`wifi: ap:` is configured: with the AP silently dead, a lost WiFi credential
would leave no way back in short of a serial reflash. `esp_reset_reason()` is
read directly instead, same value, no component in the RF path.

`BSSID` shows which access point the device actually associated with — useful
on its own with more than one AP on the same SSID, and specifically relevant
if `wifi: fast_connect` is ever turned back on, since that setting associates
with the first AP that answers rather than the strongest one.

### Buttons

| Button | Does |
|---|---|
| Restart | |
| Restart (Safe Mode) | Reboots into WiFi + OTA only — the way back in after a bad flash |
| Factory Reset | Wipes the WiFi credentials and every stored preference |

---

## How it works

### Network

Both the Home Assistant API (encrypted) and MQTT are configured. **MQTT is
off at boot** and brought up with the `MQTT` switch; `enable_on_boot: False`
and the switch's `restore_mode: ALWAYS_OFF` have to stay in step — a mismatch
would silently override one of them a moment after boot while the config
validates happily.

MQTT discovery is off: entities are already adopted through the API, and
leaving it on would create a duplicate set in Home Assistant.

> With MQTT off, `esphome logs` cannot find the device by asking the broker on
> `esphome/discover/${project_name}`. Give it the address instead:
> `esphome logs ikea-vindriktning.yaml --device ikea-vindriktning.local`

`api: reboot_timeout: 0s` and `mqtt: reboot_timeout: 0s`: this node keeps
measuring and publishing over MQTT even without an API client.

### Update interval

`update_interval` is a single substitution (`60s` by default), referenced by
every sensor that polls on a timer. `Temperature` (a `combination` sensor) and
`Absolute Humidity` recompute whenever their sources publish a new value
instead, so they have no `update_interval` option of their own.

---

## License

GPL-3.0. See `LICENSE`.
