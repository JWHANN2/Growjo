# Growjo Server

Initial server scaffold for the IoT plant monitoring system described in the handoff document.

## Scope

This repository is currently structured for the server side:

- `Mosquitto` as the MQTT broker
- `InfluxDB` as the time-series database
- `Grafana` for dashboards and KPIs
- `Telegraf` for MQTT-to-Influx ingestion

The Raspberry Pi gateway is expected to:

- read sensors locally
- add UTC timestamps and metadata
- publish JSON payloads over MQTT on the LAN

## Expected MQTT payload

Telegraf is currently configured for payloads shaped like:

```json
{
  "measurement": "soil",
  "site": "site1",
  "room": "room1",
  "plant_id": "plant1",
  "sensor_id": "soil-moisture-1",
  "value": 38.2,
  "unit": "%",
  "timestamp": "2026-01-20T19:40:00Z"
}
```

Suggested topics:

- `grow/site1/room1/plant1/moisture`
- `grow/site1/room1/environment/temp`
- `grow/test/pi01/heartbeat`
- `grow/test/pi01/system`
- `grow/test/pi01/camera`

The current Telegraf config subscribes broadly and uses the JSON body as the source of truth for measurement, tags, fields, and timestamp.

## Raspberry Pi test telemetry

Before plant sensors are installed, a Raspberry Pi can publish generic test telemetry through:

`Raspberry Pi -> Mosquitto -> Telegraf -> InfluxDB bucket plant_metrics -> Grafana`

Telegraf subscribes to these test topics:

- `grow/test/{device_id}/heartbeat`
- `grow/test/{device_id}/system`
- `grow/test/{device_id}/camera`

The JSON body is the source of truth for measurement name, tags, fields, and timestamp. Use UTC ISO 8601 timestamps where possible.

### Publish test messages

Run these examples from any machine that has `mosquitto_pub` and can reach the server. Replace `<server-ip>` with the Growjo server IP and update the timestamp if needed.

Heartbeat:

```sh
mosquitto_pub -h <server-ip> -p 1883 -t grow/test/pi01/heartbeat -m '{"measurement":"pi_heartbeat","timestamp":"2026-01-20T19:40:00Z","site":"home","room":"testbench","device_id":"pi01","status":"online","value":1}'
```

System metric:

```sh
mosquitto_pub -h <server-ip> -p 1883 -t grow/test/pi01/system -m '{"measurement":"pi_system","timestamp":"2026-01-20T19:41:00Z","site":"home","room":"testbench","device_id":"pi01","metric":"cpu_temp_c","value":52.3,"unit":"C"}'
```

Camera event:

```sh
mosquitto_pub -h <server-ip> -p 1883 -t grow/test/pi01/camera -m '{"measurement":"pi_camera","timestamp":"2026-01-20T19:42:00Z","site":"home","room":"testbench","device_id":"pi01","camera_id":"cam01","event_type":"snapshot","value":1,"image_url":"http://<pi-ip>:5000/latest.jpg"}'
```

### Grafana Flux queries

Use the provisioned InfluxDB datasource in Grafana and query the `plant_metrics` bucket.

Heartbeat status:

```flux
from(bucket: "plant_metrics")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "pi_heartbeat")
  |> filter(fn: (r) => r.device_id == "pi01")
  |> filter(fn: (r) => r._field == "value")
  |> last()
```

CPU temperature:

```flux
from(bucket: "plant_metrics")
  |> range(start: -6h)
  |> filter(fn: (r) => r._measurement == "pi_system")
  |> filter(fn: (r) => r.device_id == "pi01")
  |> filter(fn: (r) => r.metric == "cpu_temp_c")
  |> filter(fn: (r) => r._field == "value")
```

Latest camera snapshot URL:

```flux
from(bucket: "plant_metrics")
  |> range(start: -24h)
  |> filter(fn: (r) => r._measurement == "pi_camera")
  |> filter(fn: (r) => r.device_id == "pi01")
  |> filter(fn: (r) => r.camera_id == "cam01")
  |> filter(fn: (r) => r.event_type == "snapshot")
  |> filter(fn: (r) => r._field == "image_url")
  |> last()
```

## First-time setup

1. Copy `.env.example` to `.env`.
2. Replace the placeholder passwords and token.
3. Start the stack with `docker compose up -d`.
4. Open:
   - Grafana: `http://<server-ip>:3000`
   - InfluxDB: `http://<server-ip>:8086`
   - MQTT broker: `<server-ip>:1883`

## Notes

- `allow_anonymous true` is only suitable for early LAN testing. Lock this down before broader deployment.
- InfluxDB 2.x still requires an initial setup flow unless you automate it further.
- Grafana provisioning includes the InfluxDB datasource definition, but the token and bucket must exist first.
- KPI dashboards are not included yet; this is only the base server stack.

## Next server tasks

1. Add an InfluxDB bootstrap script so org, bucket, and token are created automatically.
2. Add Grafana dashboards for moisture, temperature, humidity, and light trends.
3. Add retention and downsampling policies.
4. Add MQTT auth and TLS.
5. Add a small test publisher to validate end-to-end ingestion from the server side.
