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

The current Telegraf config subscribes broadly and uses the JSON body as the source of truth for measurement, tags, fields, and timestamp.

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
