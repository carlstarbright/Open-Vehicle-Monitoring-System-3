# OVMS Nissan Leaf Improvements

**Project ID:** OVMS-LEAF  
**Started:** 2026-02-14  
**Team:** Andrew Lawson, Carl Robinson (Starbright Lab)

## Mission

Improve OVMS support for the Nissan Leaf, especially for battery-swapped vehicles. Fill the gaps left by dormant upstream development.

## Current State

- **Fork:** https://github.com/carlstarbright/Open-Vehicle-Monitoring-System-3
- **Local clone:** /Users/carl/ovms-firmware
- **Our OVMS unit:** 192.168.4.187 (firmware 3.3.004)
- **Vehicle:** Nissan Leaf with 40kWh battery swap

## Problems to Solve

### Priority 1: Monitoring & Diagnostics
- [ ] Add UDS Mode 19 DTC readout (currently unsupported)
- [ ] Expose DTCs via MQTT for automated monitoring
- [ ] Better CAN error detection (for diagnosing solder joint issues)
- [ ] 12V battery health tracking with alerts

### Priority 2: Missing Functionality  
- [ ] Fix lock/unlock commands (code exists but doesn't work)
- [ ] Proper climate control (currently mapped to random HomeLink button)
- [ ] Remote pre-conditioning that actually works

### Priority 3: Battery Swap Support
- [ ] Document CAN differences between battery generations
- [ ] BMS health analytics for swapped packs
- [ ] Cell-level degradation tracking

### Priority 4: User Experience
- [ ] Better web dashboard
- [ ] Home Assistant integration improvements
- [ ] Consider custom iOS/Android app (current OVMS app is garbage)

## Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Nissan     │────▶│    OVMS     │────▶│  Mosquitto  │
│  Leaf CAN   │     │   ESP32     │     │  (Mac mini) │
└─────────────┘     └─────────────┘     └─────────────┘
                           │                    │
                           ▼                    ▼
                    ┌─────────────┐     ┌─────────────┐
                    │ OVMS Cloud  │     │ Daily Logs  │
                    │  (V2 API)   │     │  (JSONL)    │
                    └─────────────┘     └─────────────┘
                                               │
                                               ▼
                                        ┌─────────────┐
                                        │ Daily Cron  │
                                        │  Analysis   │
                                        └─────────────┘
```

## Development Setup

### Prerequisites
- ESP-IDF v3.3.4 (specific version required)
- Python 3.x
- USB-serial adapter for flashing

### Building
```bash
cd /Users/carl/ovms-firmware/vehicle/OVMS.V3
# Setup ESP-IDF environment
source /path/to/esp-idf/export.sh
# Configure
idf.py menuconfig
# Build
idf.py build
```

### Testing
- Use MQTT logging to capture before/after behavior
- Compare CAN traffic with OBD2 scan tools
- Document all CAN message IDs and their meanings

## Key Files

- `vehicle/OVMS.V3/components/vehicle_nissanleaf/src/vehicle_nissanleaf.cpp` - Main vehicle logic (~104KB)
- `vehicle/OVMS.V3/components/vehicle_nissanleaf/src/vehicle_nissanleaf.h` - Class definition
- `vehicle/OVMS.V3/components/vehicle_nissanleaf/src/nl_web.cpp` - Web interface

## CAN Bus Reference

### Known ECU Addresses
| ECU | TX ID | RX ID | Description |
|-----|-------|-------|-------------|
| LBC (EV-ECU) | 0x79B | 0x7BB | Battery management |
| VCM | 0x797 | 0x7B7 | Vehicle control |
| BCM | 0x743 | 0x763 | Body control |
| ABS | 0x740 | 0x760 | Brakes |

### OBD2 Modes Supported
- 01: Current data
- 02: Freeze frame
- 09: Vehicle info  
- 10: Diagnostic session
- 1A: Read ECU ID
- 21: Manufacturer specific (Nissan)
- 22: Manufacturer specific (extended)

### Missing (TODO)
- 19: Read DTCs (UDS) - **HIGH PRIORITY**
- 14: Clear DTCs

## Monitoring Infrastructure

### MQTT Logging
- **Script:** /Users/carl/clawd/scripts/ovms-mqtt-logger.sh
- **Logs:** /Users/carl/clawd/data/ovms/logs/ovms-YYYY-MM-DD.jsonl
- **Service:** com.starbright.ovms-logger (launchd)

### Daily Analysis (8am EST)
- Reviews previous day's logs
- Alerts on: 12V drops, cell imbalance, CAN errors
- Updates battery health trends in memory

## Resources

- [OVMS User Guide](https://docs.openvehicles.com/)
- [Nissan Leaf CAN Bus Wiki](https://www.mynissanleaf.com/wiki/index.php?title=CAN_bus)
- [Dala's Battery Emulator](https://github.com/dalathegreat/Nissan-LEAF-Battery-Upgrade)
- [ESP-IDF Documentation](https://docs.espressif.com/projects/esp-idf/)

## Log

- 2026-02-14: Project created, forked repo, set up MQTT logging
