# OVMS Build Setup

**Last Updated:** 2026-02-15

## Quick Start (Docker - Recommended)

The easiest way to build OVMS firmware is using Docker, which handles all ESP-IDF dependencies:

```bash
# Build the Docker image (first time only)
cd /Users/carl/ovms-firmware
docker build -t ovms-builder -f docker/Dockerfile .

# Run interactive shell
docker run -it --rm -v $(pwd):/project ovms-builder

# Inside container:
cd /project/vehicle/OVMS.V3
cp support/sdkconfig.default.hw31 sdkconfig
make -j4
```

## Native macOS Setup (Advanced)

Native builds on macOS require ESP-IDF v3.3.4, which has compatibility issues with modern macOS (especially Apple Silicon). Docker is strongly recommended.

### If you must build natively:

1. **Install ESP-IDF v3.3.4:**
```bash
mkdir -p ~/esp
cd ~/esp
git clone -b v3.3.4 --recursive https://github.com/espressif/esp-idf.git
```

2. **Install toolchain:**
```bash
# Download from: https://dl.espressif.com/dl/xtensa-esp32-elf-osx-1.22.0-96-g2852398-5.2.0.tar.gz
tar -xzf xtensa-esp32-elf-*.tar.gz -C ~/esp/
```

3. **Create Python venv:**
```bash
python3 -m venv ~/esp/esp-idf-venv
source ~/esp/esp-idf-venv/bin/activate
pip install -r ~/esp/esp-idf/requirements.txt
```

4. **Set environment:**
```bash
export IDF_PATH=~/esp/esp-idf
export PATH=~/esp/xtensa-esp32-elf/bin:$PATH
source ~/esp/esp-idf-venv/bin/activate
```

5. **Build:**
```bash
cd /Users/carl/ovms-firmware/vehicle/OVMS.V3
cp support/sdkconfig.default.hw31 sdkconfig
make -j4
```

### Known Issues

- **ncurses errors:** Homebrew's ncurses may not link correctly. Install system ncurses or use Docker.
- **Python pkg_resources:** Install `pip install setuptools` in the venv.
- **Apple Silicon:** x86 toolchain runs via Rosetta but may have issues.

## Configuration Files

| Config | Description |
|--------|-------------|
| `sdkconfig.default.hw31` | OVMS v3.1 hardware |
| `sdkconfig.default.hw30` | OVMS v3.0 hardware |
| `sdkconfig.bluetooth.hw31` | v3.1 with Bluetooth |
| `sdkconfig.lilygo_tc` | TTGO/Lilygo board |

## Flashing

```bash
# Connect OVMS via USB
make flash

# Or specify port
make flash ESPPORT=/dev/tty.usbserial-*
```

## OTA Updates

You can also flash via the OVMS web interface:
1. Build firmware: `make`
2. Go to OVMS web UI → Config → Firmware → Flash from file
3. Upload `build/ovms3.bin`

## Project Structure

```
vehicle/OVMS.V3/
├── components/
│   ├── vehicle_nissanleaf/    ← Our focus
│   │   └── src/
│   │       ├── vehicle_nissanleaf.cpp
│   │       └── vehicle_nissanleaf.h
│   └── ... (other vehicles)
├── main/
├── support/
│   └── sdkconfig.*           ← Build configs
└── build/
    └── ovms3.bin             ← Output firmware
```
