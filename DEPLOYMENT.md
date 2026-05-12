# RS-485 to WiFi Bridge Deployment Guide

Simple deployment guide for ESP32-C3 RS-485 to WiFi bridge device.

## 1. Pre-Deployment Checklist

### Hardware Verification
- [ ] ESP32-C3 powers on at 3.3V
- [ ] 10µF and 100nF decoupling capacitors installed near VCC/GND
- [ ] RS-485 module (MAX3485) connected: A/B lines to GPIO1 (TX), GPIO3 (RX)
- [ ] USB-C connected for programming
- [ ] Serial console visible at ~115200 baud

### Pin Connections (See hardware-config.md)
| Component | GPIO | Purpose |
|-----------|------|---------|
| RS-485 TX | 1 | UART transmit to MAX3485 |
| RS-485 RX | 3 | UART receive from MAX3485 |
| Status LED | 10 | Power/status indicator |
| User Button | 11 | Control (optional) |

### Serial Console Test
```bash
# Verify USB device appears
ls /dev/tty* | grep -i usb  # Linux
ls /dev/tty.* | grep -i usb  # macOS

# Quick connectivity test
screen /dev/ttyUSB0 115200
# Should show boot messages
```

---

## 2. Secrets Configuration

### Create secrets.yaml
```bash
cp secrets-template.yaml secrets.yaml
```

### Edit secrets.yaml
```yaml
wifi_ssid: "YourWiFi"
wifi_password: "YourPassword"
wifi_fallback_ap_password: "FallbackPassword"
mqtt_broker: "192.168.1.100"
mqtt_user: "rs485_bridge"
mqtt_password: "SecurePassword"
api_encryption_key: "$(python3 -c 'import secrets; print(secrets.token_hex(32))')"
api_password: "APIPassword"
ota_password: "OTAPassword"
```

### Generate Keys
```bash
# Encryption and OTA passwords
python3 -c "import secrets; print(secrets.token_hex(32))"
```

---

## 3. ESPHome Setup

### Install
```bash
pip install --upgrade esphome
esphome version
```

### Validate YAML
```bash
esphome config esphome-mqtt.yaml
```

---

## 4. Firmware Compilation & Upload

### Compile
```bash
esphome compile esphome-mqtt.yaml
```

### Upload via USB
```bash
esphome upload esphome-mqtt.yaml
```

### Monitor Logs
```bash
esphome logs esphome-mqtt.yaml
```

Expected boot sequence:
```
[I][esphome]: ESP-IDF version: ...
[I][wifi]: Connecting to SSID...
[I][wifi]: Connected to WiFi
[I][mqtt]: Connecting to MQTT...
[I][mqtt]: MQTT Connected!
```

---

## 5. Initial Device Testing

### Phase 1: Boot & Connectivity
1. Power device on
2. Check serial logs show WiFi + MQTT connection
3. Verify device connects within 1 minute

### Phase 2: RS-485 Communication
1. Connect RS-485 slave device (A/B terminals)
2. Check logs for UART data:
   ```bash
   esphome logs esphome-mqtt.yaml | grep -i "uart\|modbus"
   ```
3. Verify data appearing in MQTT topics

### Phase 3: MQTT Data Flow
Monitor MQTT traffic:
```bash
mosquitto_sub -h 192.168.1.100 -u rs485_bridge -P <password> \
  -t "home/#" -v
```

Expected topics (depends on your slave device):
```
home/rs485/raw_data
home/rs485/parsed_values
home/device/uptime
```

### Phase 4: Stability Test
Let device run for 30 minutes:
- Monitor for connection drops
- Check heap memory stays stable
- Verify MQTT messages publishing regularly

---

## 6. Debugging

### WiFi Issues
```bash
# Check WiFi signal strength in logs
esphome logs esphome-mqtt.yaml | grep -i "signal\|rssi"

# Try fallback AP
# Connect to: esphome-water-meter
# Password: wifi_fallback_ap_password from secrets.yaml
```

### MQTT Connection Failed
```bash
# Verify broker running
mosquitto_pub -h 192.168.1.100 -u rs485_bridge -P <password> \
  -t test -m "hello"

# Check credentials in secrets.yaml
# Verify firewall allows port 1883
```

### No RS-485 Data
```bash
# Check UART logs
esphome logs esphome-mqtt.yaml | grep -i "uart"

# Verify wiring: A/B lines connected
# Test with continuity meter
# Check slave device is powered and responding
```

### Device Crashes
```bash
# Check logs for errors
esphome logs esphome-mqtt.yaml | grep "\[E\]"

# Monitor heap usage
esphome logs esphome-mqtt.yaml | grep "heap"
```

---

## 7. OTA Updates

Once device is stable and on WiFi:

```bash
# Flash new firmware over-the-air
esphome upload esphome-mqtt.yaml --device 192.168.1.XXX
```

---

## 8. Production Checklist

- [ ] Device boots without errors
- [ ] WiFi connects reliably (< 30 seconds)
- [ ] MQTT connects and stays connected
- [ ] RS-485 data flowing to MQTT
- [ ] 30-minute stability test passed
- [ ] MQTT messages retain on broker
- [ ] Backup firmware binary saved
- [ ] OTA updates tested

---

## Reference

**Hardware**: [hardware-config.md](hardware-config.md)
**MQTT Setup**: [MQTT-SETUP.md](MQTT-SETUP.md)
**ESPHome**: https://esphome.io/

