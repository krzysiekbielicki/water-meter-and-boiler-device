# ESPHome MQTT Configuration Setup Guide

## Overview
This document explains the MQTT configuration for the ESP32-C3 water meter and boiler device.

## Files
- **esphome-mqtt.yaml** - Main MQTT and WiFi configuration with template sensors
- **secrets-template.yaml** - Template for required secrets (copy to secrets.yaml)

## Quick Setup

### 1. Create secrets.yaml
```bash
cp secrets-template.yaml secrets.yaml
```

Edit `secrets.yaml` with your values:
- WiFi SSID and password
- MQTT broker IP (e.g., 192.168.1.100)
- MQTT username and password
- API and OTA passwords

### 2. MQTT Broker Requirements
- Running on port 1883 (plain MQTT, not TLS)
- Has user authentication enabled (username: mqtt_user)
- Accessible from your WiFi network

### 3. Device LED Indicators
- **Rapidly blinking** - Connecting to WiFi/MQTT
- **Slow blinking** - WiFi connected, MQTT connecting
- **Solid on** - Fully connected (WiFi + MQTT + API)
- **Off** - No connection

## MQTT Topic Structure

### Water Meter Topics
```
home/water-meter/consumption    → Float value in m³ (retained)
home/water-meter/flow           → Float value in L/min (retained)
```

### Boiler Topics
```
home/boiler/temperature         → Float value in °C (retained)
home/boiler/status              → "on" or "off" (retained)
home/boiler/setpoint            → Float value in °C (retained)
```

### Device Status Topics
```
home/device/status              → "online" or "offline" (retained)
home/device/uptime              → Integer seconds (retained)
home/device/wifi_signal         → Integer dBm (retained)
home/device/ip_address          → String IP address (retained)
```

## Testing MQTT Connectivity

### Subscribe to all device topics:
```bash
mosquitto_sub -h 192.168.1.100 -u mqtt_user -P mqtt_password -t "home/#" -v
```

### Monitor device status only:
```bash
mosquitto_sub -h 192.168.1.100 -t "home/device/status" -v
```

### Monitor water meter consumption:
```bash
mosquitto_sub -h 192.168.1.100 -t "home/water-meter/consumption" -v
```

### Test bidirectional communication:
```bash
mosquitto_pub -h 192.168.1.100 -t "home/test" -m "Hello from broker"
```

## Home Assistant Integration

The configuration includes Home Assistant MQTT Discovery. Once connected:
1. Enable MQTT in Home Assistant
2. Configure discovery prefix as "homeassistant"
3. Device should auto-discover in HA

## Troubleshooting

### "MQTT disconnected" in logs
- Verify `mqtt_broker` IP in secrets.yaml
- Check MQTT broker is running: `mosquitto -d -p 1883`
- Test connectivity: `telnet 192.168.1.100 1883`
- Verify username/password are correct

### WiFi keeps disconnecting
- Check WiFi signal strength (should be > -80 dBm)
- Reduce `power_save_mode` interference
- Place device closer to WiFi router

### No data published to MQTT
- Check device connects to WiFi (look for "WiFi connected" in logs)
- Verify MQTT broker is accessible
- Enable DEBUG logging: change `logger: level: DEBUG`
- Check sensors are properly configured with actual hardware

### Connection intermittent
- Increase `keepalive: 60s` interval if needed
- Check for WiFi interference (change channel in WiFi settings)
- Monitor WiFi signal strength in `home/device/wifi_signal`

## Integration with Main Configuration

To use this configuration in your main esphome.yaml:

```yaml
esphome:
  name: water-meter-boiler

# Import MQTT configuration
<<: !include esphome-mqtt.yaml

# Add additional components here...
```

Or use ESPHome dashboard to select this configuration file directly.

## Next Steps

1. Replace placeholder sensor readings with actual hardware integration:
   - Water meter via pulse counter or direct sensor
   - Boiler temperature via DS18B20, DHT22, etc.
   - Boiler status via GPIO relay monitoring

2. Connect actual sensors to appropriate GPIO pins

3. Configure sensor-specific parameters (pulse count conversion, calibration, etc.)

4. Test OTA updates: ESPHome automatically handles firmware updates

## Security Notes

- Store secrets.yaml securely (not in git)
- Change default API and OTA passwords
- Consider using TLS for remote access (requires certificate setup)
- Restrict MQTT broker access to internal network only
