# Flowis+ Water Meter Integration (WMBus)

## Overview

This guide explains how to configure ESPHome to read your **Flowis+ water meter** via **WMBus (Wireless Meter-Bus)** protocol.

**Key Points:**
- Flowis+ transmits meter data via WMBus at 868 MHz
- CC1101 RF module receives WMBus frames
- You need your meter's ID to filter and read correct data
- Configuration file: `esphome-cc1101-wmbus.yaml`

## Finding Your Meter ID

**Method 1: Check the meter label**
1. Locate the physical meter (usually in basement or water supply inlet)
2. Look for an 8-digit hexadecimal number on the label or documentation
3. Format: `0x12AB34CD` or just `12AB34CD`
4. This is your **Meter ID (MBUS ID)**

**Method 2: Capture from logs (if you don't have the ID)**
1. Flash the device with WMBus config
2. In ESPHome logs, watch for incoming WMBus frames:
   ```
   [D] wmbus: Received frame from Meter 0x12AB34CD
   ```
3. Note all meter IDs you receive - verify which is yours
4. Update `meter_id` field and reflash

**Method 3: Contact your water company**
- Provide your meter serial number
- They can supply the WMBus ID

## Configuration Steps

### Step 1: Update Meter ID

Edit `esphome-cc1101-wmbus.yaml` and replace all occurrences of `0xXXXXXXXX` with your actual meter ID:

```yaml
wmbus:
  meter_id: 0x12AB34CD  # Replace with your ID

sensor:
  - platform: wmbus
    meter_id: 0x12AB34CD  # Replace with your ID
```

### Step 2: Check for Encryption

**Is your meter encrypted?**

Most consumer Flowis+ meters are **not encrypted** by default. If you suspect encryption:

1. Check meter documentation for AES-128 key
2. Uncomment and update the key line:
   ```yaml
   wmbus:
     # key: "0000000000000000000000000000000000000000000000000000000000000000"
   ```
3. Replace with actual 64-character hex key if known

### Step 3: Flash to Device

```bash
# Validate
esphome validate esphome-cc1101-wmbus.yaml

# Compile
esphome compile esphome-cc1101-wmbus.yaml

# Upload
esphome upload esphome-cc1101-wmbus.yaml --device AUTO

# Monitor logs
esphome logs esphome-cc1101-wmbus.yaml
```

### Step 4: Verify Reception

Watch the logs for:
```
[D] wmbus: Received WMBus frame from meter 0x12AB34CD
[D] wmbus: Total water: 123.456 m³
[D] wmbus: Current flow: 0.001 m³/h
```

## WMBus Frame Structure

Flowis+ sends periodic frames with:
- **Total consumption** (m³) - current meter reading
- **Current flow** (m³/h) - real-time flow rate
- **Volume liquid** (L) - additional consumption data
- **Leak detection** - if connected plumbing leaks
- **Device status** - meter health indicators
- **Timestamp** - when data was recorded

## Transmission Details

| Parameter | Value |
|-----------|-------|
| Frequency | 868 MHz (868.95 MHz nominal) |
| Protocol | WMBus EN13757-4 |
| Mode | T1 (transmit-only from meter perspective) |
| Interval | Every 8-12 seconds (configurable, affects battery) |
| Range | Up to 500m (open field) |
| Power | 16-25 mW |
| Encryption | AES-128 (optional, modes 5 & 7) |

## Troubleshooting

### No frames received
1. **Distance:** Move device closer to meter (try <10m first)
2. **Antenna:** Ensure CC1101 antenna is properly attached
3. **Frequency:** Verify meter is 868 MHz (not 915 MHz)
4. **Meter ID:** Check you have correct ID format (must be 8-digit hex)
5. **Logs:** Check ESPHome debug logs for errors
   ```bash
   esphome logs esphome-cc1101-wmbus.yaml
   ```

### Corrupted/Invalid data
1. Check Modbus CRC errors in logs
2. Verify SPI connection (CLK, MOSI, MISO, CS pins)
3. Try adjusting SPI frequency (lower to 2.5MHz if errors persist)
4. Ensure decoupling capacitors on CC1101 power pins

### Encryption errors
1. If logs show "Encryption failed," the meter may use encryption
2. Obtain AES-128 key from meter provider or water company
3. Update config with key and reflash

## MQTT Topics

Sensor data published to:
- `home/water-meter/consumption` → Current reading (m³)
- `home/water-meter/flow` → Current flow (m³/h)
- `home/water-meter/last_flow` → Recent flow (L)
- `home/water-meter/leak_detected` → Leak alarm (on/off)
- `home/water-meter/meter_error` → Error status (on/off)

## Hardware Reference

| Component | GPIO |
|-----------|------|
| CC1101 CS | GPIO4 |
| CC1101 CLK | GPIO5 |
| CC1101 MISO | GPIO6 |
| CC1101 MOSI | GPIO7 |
| RGB LED | GPIO8 |

See `hardware-config.md` for complete pinout.

## References

- [Flowis+ Datasheet](https://www.aquabuilding.com/)
- [WMBus EN13757-4 Standard](https://www.dlms.com/)
- [ESPHome WMBus Component](https://esphome.io/components/wmbus.html)
- [CC1101 Datasheet](https://www.ti.com/product/CC1101)

## Support

If you encounter issues:
1. Enable DEBUG logging in esphome.yaml
2. Check serial output for WMBus frame details
3. Verify meter ID matches received frames
4. Confirm CC1101 antenna connection
5. Try moving closer to meter for testing
