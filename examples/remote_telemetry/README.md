# Remote Telemetry Node

This example targets ESP32-based boards with WiFi (for example the Seeed XIAO ESP32S3 WIO, Heltec LoRa32 V3, and LilyGO T-Beam SX1276) and turns the device into a WiFi-enabled telemetry forwarder. The firmware logs in to configured repeaters, fetches telemetry data and republishes it to an MQTT broker.

## Configuration

- On first boot the node starts the WiFiManager portal that captures WiFi, MQTT credentials, telemetry topics and repeater details and saves them to `/telemetry.json` on SPIFFS.
- Populate `examples/remote_telemetry/credentials.h` with deployment defaults for the MQTT host, username and password; leaving those fields blank in the portal applies these values automatically.
- `credentials.h` also sets `FIRMWARE_UPDATE_URL`, the HTTPS location the node checks in response to the `update_firmware` control command (see below). It must serve the `.bin` built for this exact board/environment. The node does not validate the server's certificate (`WiFiClientSecure::setInsecure()`), so this only protects the transfer from passive eavesdropping, not from a spoofed or compromised host.
- To change settings later, trigger the configuration portal (e.g. erase `/telemetry.json`) or push updates over MQTT as described below.

## Building

Select an ESP32 WiFi environment such as `env:Xiao_S3_WIO_remote_telemetry`, `env:Heltec_v3_remote_telemetry`, or `env:Tbeam_SX1276_remote_telemetry` from PlatformIO, or run:

```
platformio run -e Xiao_S3_WIO_remote_telemetry
```

or

```
platformio run -e Tbeam_SX1276_remote_telemetry
```

or

```
platformio run -e Heltec_v3_remote_telemetry -t mergebin
```

If `.pio/libdeps` is cleaned, reapply the CayenneLPP ArduinoJson patch as noted in the build logs (switch `add<JsonObject>()` to `createNestedObject()` in CayenneLPP.cpp) and ensure `densaugeo/base64` and `ArduinoJson` stay in `lib_deps`.

## Logging

Runtime logging uses `Serial` at 115200 baud. Set the build flag `-D REMOTE_TELEMETRY_DEBUG=0` to silence all debug output for production deployments.

## MQTT Control & Payloads

The node listens for JSON commands on the configured control topic and publishes telemetry and status messages to their respective topics. At runtime the control topic gains a suffix with the first two bytes of the node's public key (e.g. `telemetry/control/364C`) so multiple nodes can share a base topic without collisions.

### Commands to the Node

Send JSON payloads to the MQTT control topic (`settings.mqttControlTopic`). Key operations:

#### Repeater management

```json
{"command": "list_repeaters"}
```

> The `repeaters_snapshot` response trims each entry down to `name` and a 2-byte `pubKey` prefix (4 hex chars) to keep the message within the MQTT client's packet size limit. Use `get_repeater` below to resolve a prefix to its full record.

```json
{"command": "get_repeater", "pubKey": "364c"}
```

> Looks up repeaters whose `pubKey` starts with the given hex prefix (2 to 64 characters, i.e. 1 to 32 bytes). Responds on the status topic with the full `pubKey`, `name`, and `password` for every match — this is a targeted, small response, so unlike `repeaters_snapshot` it isn't trimmed. If two configured repeaters happen to share the same prefix, both are returned; use more hex digits to narrow it down further.

```json
{
  "command": "add_repeater",
  "repeater": {
    "name": "IT-Orp-S omni",
    "password": "",
    "pubKey": "8f3e2e611bf8469deed5ece8a59249281bca7e891febdd58861a406fbf73a8bc"
  }
}
```

```json
{
  "command": "update_repeater",
  "repeater": {
    "pubKey": "8f3e2e611bf8469deed5ece8a59249281bca7e891febdd58861a406fbf73a8bc",
    "password": "",
    "name": "IT-Orp-S omni"
  }
}
```

```json
{
  "command": "remove_repeater",
  "pubKey": "8f3e2e611bf8469deed5ece8a59249281bca7e891febdd58861a406fbf73a8bc"
}
```

#### Firmware update

```json
{"command": "update_firmware"}
```

> The node sends a `HEAD` request to `FIRMWARE_UPDATE_URL` (from `credentials.h`) and compares the response's `Last-Modified` header to the value it last installed. It always responds with a `firmware_update_check` status event *before* downloading anything:
>
> ```json
> {
>   "event": "firmware_update_check",
>   "decision": "accepted",
>   "reason": "changed",
>   "currentVersion": "Tue, 01 Jul 2025 10:00:00 GMT",
>   "remoteVersion": "Wed, 24 Sep 2025 14:22:10 GMT",
>   "uptimeMs": 123456
> }
> ```
>
> `decision` is `accepted` (`reason` is `changed`, or `first_check` if nothing was installed yet), `declined` (`reason: unchanged` — the header matches what's already installed, no download), or `error` (`reason: check_failed` or `url_not_configured` — the host couldn't be reached, or returned no `Last-Modified` header; treated the same as declined, nothing is downloaded).
>
> Only on `accepted` does the node download and flash the `.bin` at that URL to the inactive OTA partition. On success it persists the new `Last-Modified` value and reboots (a `boot` status event on reconnect confirms it came back up); on failure it publishes a `firmware_update_failed` status event and keeps running the current firmware untouched — a failed or interrupted download cannot leave the device unbootable, since the new partition is only marked bootable after the image is fully validated.
>
> The URL must be HTTPS and serve the `.bin` built for this exact board/environment; nothing here checks that the served binary matches the running hardware. The node connects with `WiFiClientSecure::setInsecure()` — the transfer is encrypted but the server's certificate is not validated.

#### Reboot

```json
{"command": "reboot"}
```

> Acknowledged with a `rebooting` status event, then the node calls `esp_restart()` immediately. `restart` is accepted as an alias.

#### Runtime tuning

```json
{"pollIntervalSeconds": 900}
```

```json
{"timeoutRetrySeconds": 120}
```

```json
{"loginRetrySeconds": 600}
```

```json
{"telemetryTopic": "meshcore/telemetry"}
```

> The telemetry topic update takes effect immediately and is persisted to `/telemetry.json`. The `mqttTelemetryTopic` key is accepted as an alias.

### Responses from the Node

Status messages land on `settings.mqttStatusTopic`, for example:

```json
{
  "event": "repeater_added",
  "detail": "RemoteSite",
  "uptimeMs": 52341,
  "pollIntervalMs": 1800000,
  "timeoutRetryMs": 30000,
  "loginRetryMs": 120000,
  "topics": {
    "telemetry": "meshcore/field/telemetry",
    "status": "meshcore/field/status",
    "control": "meshcore/field/control/364C"
  },
  "node": {
    "pubKey": "8f3e2e611bf8469deed5ece8a59249281bca7e891febdd58861a406fbf73a8bc"
  }
}
```

```json
{
  "event": "telemetry_topic_updated",
  "detail": "meshcore/field/telemetry",
  "uptimeMs": 54012
}
```

```json
{
  "event": "repeater_detail",
  "query": "364c",
  "matches": 1,
  "repeaters": [
    {
      "name": "IT-Orp-S omni",
      "password": "",
      "pubKey": "8f3e2e611bf8469deed5ece8a59249281bca7e891febdd58861a406fbf73a8bc"
    }
  ],
  "node": {
    "pubKey": "8f3e2e611bf8469deed5ece8a59249281bca7e891febdd58861a406fbf73a8bc"
  }
}
```

> `get_repeater` responses include the full `password`, unlike `repeaters_snapshot`. Only send this on a private, trusted broker/network.

### Telemetry Payloads

Telemetry is published to the current telemetry topic using the normalised format:

```json
{
  "type": "telemetry",
  "repeater": "IT-Orp-S omni",
  "metrics": {
    "voltage": 4.05,
    "current": 0.009,
    "power": 0.0
  },
  "samples": [
    {"channel": 1, "type": 116, "value": 4.0},
    {"channel": 2, "type": 116, "value": 4.05},
    {"channel": 2, "type": 117, "value": 0.009},
    {"channel": 2, "type": 128, "value": 0.0}
  ],
  "collectedAt": "2026-01-11T15:31:43.408Z",
  "collectedAtText": "15:31 on 11/01/2026"
}
```

`metrics` captures the most recent INA219-style voltage/current/power values when available, while `samples` enumerates the raw Cayenne LPP records forwarded by the repeater.

## Operational Notes

- After NTP synchronisation the node schedules a daily reboot in the 03:00 local-time window to minimise disruption.
- Guest logins are issued first to discover a direct path; the stored admin credentials are only sent once a unicast route is known.
