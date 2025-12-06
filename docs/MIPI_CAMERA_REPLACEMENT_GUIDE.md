# RunCam WiFiLink2 - MIPI Camera Replacement Guide

## Overview

This guide provides step-by-step instructions for replacing the built-in camera in RunCam WiFiLink2 with another MIPI video source. The RunCam WiFiLink2 typically uses a SigmaStar SoC (SSC009 or Infinity6 series), which supports MIPI CSI-2 camera interfaces.

## Prerequisites

- RunCam WiFiLink2 device
- Supported MIPI camera sensor (see supported sensors list below)
- Soldering equipment (if hardware modifications are needed)
- USB-TTL serial adapter for debugging
- Basic Linux command line knowledge
- OpenIPC firmware build environment

## Supported MIPI Camera Sensors

The SigmaStar Infinity6 platform currently supports the following MIPI sensors:

- **GalaxyCore GC2053** - 2MP MIPI sensor
- **Sony IMX307** - 2MP MIPI sensor  
- **SmartSens SC3335** - 3MP MIPI sensor
- **SmartSens SC2239** - 2MP MIPI/Parallel sensor
- **SmartSens SC2335** - 2MP MIPI/Parallel sensor

Additional sensors may be available in the [OpenIPC sensors repository](https://github.com/openipc/sensors).

### Alternative: HDMI to MIPI Adapters

You can also use **HDMI to MIPI CSI-2 bridge chips** to convert HDMI video sources:

- **Toshiba TC358743XBG** - HDMI 1.4 to MIPI CSI-2 bridge (up to 1080p60)
- **Toshiba TC358840XBG** - HDMI 2.0 to MIPI CSI-2 bridge (up to 4K30)

This allows using any HDMI source (cameras, computers, game consoles) as input. See the [TC358743 Integration Example](EXAMPLE_TC358743_HDMI_TO_MIPI.md) for detailed instructions.

## Architecture Overview

The camera sensor integration in OpenIPC consists of several layers:

1. **Kernel Driver** - Low-level sensor driver (`.ko` module)
2. **Sensor Configuration** - Binary configuration files (`.bin`)
3. **Sensor Detection** - Auto-detection and initialization script
4. **ISP Configuration** - Image Signal Processor settings
5. **Application Layer** - Majestic streamer configuration

## Step-by-Step Replacement Guide

### Step 1: Identify Your Hardware

1. Connect to your RunCam WiFiLink2 via serial console or SSH
2. Run the following command to identify your SoC:
   ```bash
   ipcinfo -c
   ```

3. Check the current sensor:
   ```bash
   ipcinfo -s
   ```

4. Verify the vendor:
   ```bash
   ipcinfo -v
   ```

Expected output for SigmaStar-based WiFiLink2:
- SoC: `ssc009b-s01a` or similar Infinity6 variant
- Vendor: `sigmastar`

### Step 2: Hardware Connection

#### MIPI CSI-2 Interface Pinout

Ensure your new MIPI camera sensor connects to the following signals:

| Signal | Description | Notes |
|--------|-------------|-------|
| MIPI_CLK_P/N | MIPI clock differential pair | High-speed clock |
| MIPI_DATA0_P/N | Data lane 0 differential pair | Typically required |
| MIPI_DATA1_P/N | Data lane 1 differential pair | Optional, for higher bandwidth |
| I2C_SCL | I2C clock for sensor control | Usually shared I2C bus |
| I2C_SDA | I2C data for sensor control | Usually shared I2C bus |
| MCLK | Master clock output to sensor | Usually 27MHz or 24MHz |
| RESET | Sensor reset (GPIO) | Active low |
| PWDN | Power down (GPIO) | Active high/low depending on sensor |
| VDD | Power supply | Usually 3.3V or 1.8V |
| GND | Ground | Multiple ground pins |

**Important Notes:**
- MIPI signals are high-speed differential pairs - keep traces short and matched
- I2C address must not conflict with other devices on the bus
- Check sensor datasheet for power sequencing requirements
- Some sensors require separate AVDD (analog) and DVDD (digital) supplies

### Step 3: Add Sensor Driver Support

If your sensor is not in the supported list above, you need to add driver support:

#### Option A: Use Existing Sensor Driver (Recommended)

1. Check if your sensor is available in the sensors repository:
   ```bash
   # On your build machine
   cd /path/to/openipc/firmware
   ls general/package/sigmastar-osdrv-sensors/
   ```

2. If available, ensure the `sigmastar-osdrv-sensors` package is enabled in your build configuration.

#### Option B: Add Custom Sensor Driver

1. **Obtain the sensor driver** (.ko file) from:
   - Sensor manufacturer
   - OpenIPC community
   - Reference implementation from similar sensor

2. **Place the driver in the appropriate location:**
   ```bash
   # The driver should be named: sensor_SENSORNAME_mipi.ko
   # Example: sensor_imx415_mipi.ko
   /lib/modules/4.9.84/sigmastar/sensor_SENSORNAME_mipi.ko
   ```

3. **Add sensor configuration binary:**
   ```bash
   # Sensor configuration file
   /usr/bin/SENSORNAME.bin
   ```

### Step 4: Modify Sensor Loading Script

Edit the sensor loading script to include your new sensor:

```bash
vi /usr/sbin/load_sigmastar
```

Locate the `set_sensor()` function and add your sensor:

```bash
set_sensor() {
    case $SENSOR in
        gc2053|imx307|sc3335)
            insmod $MODULE/sensor_${SENSOR}_mipi.ko chmap=1
            ;;
        sc2239|sc2335)
            [ "$(fw_printenv -n soc)" = "ssc325de" ] && IFACE=parl
            insmod $MODULE/sensor_${SENSOR}_${IFACE:-mipi}.ko chmap=1
            ;;
        # Add your new sensor here
        imx415|your_sensor_name)
            insmod $MODULE/sensor_${SENSOR}_mipi.ko chmap=1
            ;;
        *)
            echo -e "\n\e[1;31mUNSUPPORTED sensor - $SENSOR\e[0m\n" | logger -s -t OpenIPC
            ;;
    esac
}
```

**In the firmware source**, modify:
```
/home/runner/work/firmware/firmware/general/package/sigmastar-osdrv-infinity6/files/script/load_sigmastar
```

### Step 5: Configure Sensor in U-Boot Environment

Set the sensor name in the U-Boot environment:

```bash
fw_setenv sensor your_sensor_name
```

Example for IMX307:
```bash
fw_setenv sensor imx307
```

To enable auto-detection instead:
```bash
fw_setenv sensor ""
```

### Step 6: Configure GPIO for Sensor Reset/Power

Some sensors require GPIO configuration for reset and power control.

1. **Identify GPIO pins** - Check the SoC datasheet and your hardware schematic

2. **Configure GPIO in the sensor driver** or create a startup script:

```bash
# Example: /etc/init.d/S36sensor-gpio
#!/bin/sh

case "$1" in
    start)
        echo "Configuring sensor GPIOs..."
        
        # Export GPIO (example: GPIO 60 for reset)
        echo 60 > /sys/class/gpio/export
        echo out > /sys/class/gpio/gpio60/direction
        
        # Reset sequence
        echo 0 > /sys/class/gpio/gpio60/value
        usleep 10000
        echo 1 > /sys/class/gpio/gpio60/value
        usleep 10000
        ;;
    stop)
        echo 60 > /sys/class/gpio/unexport
        ;;
esac
```

### Step 7: Configure Majestic Streamer

The Majestic streamer needs to know about your sensor's capabilities.

Edit `/etc/majestic.yaml`:

```yaml
system:
  sensor: your_sensor_name
  
isp:
  # Adjust based on your sensor
  slowShutter: true
  exposure: 50
  
image:
  # Resolution based on sensor capability
  width: 1920
  height: 1080
  
video0:
  codec: h264
  # Ensure resolution matches sensor output
  width: 1920
  height: 1080
  fps: 30
```

### Step 8: Test and Verify

1. **Reboot the device:**
   ```bash
   reboot
   ```

2. **Check sensor detection:**
   ```bash
   dmesg | grep sensor
   ipcinfo -s
   ```

3. **Verify sensor driver is loaded:**
   ```bash
   lsmod | grep sensor
   ```

4. **Check video stream:**
   ```bash
   curl http://device-ip/api/v1/video
   ```

5. **View RTSP stream** using VLC or similar:
   ```
   rtsp://device-ip:554/stream=0
   ```

## Building Custom Firmware

To build firmware with your custom sensor support:

### Step 1: Set Up Build Environment

```bash
git clone https://github.com/openipc/firmware.git
cd firmware
```

### Step 2: Select Configuration

For RunCam WiFiLink2 (typically SSC009B):

```bash
# For Infinity6 SSC009B
make infinity6-ssc009b_lite_defconfig
```

Or for other Infinity6 variants:
```bash
# List available configs
ls br-ext-chip-sigmastar/configs/

# Example for SSC009A
make infinity6-ssc009a_lite_defconfig
```

### Step 3: Configure Buildroot

```bash
make menuconfig
```

Navigate to:
- `External options` → Enable `sigmastar-osdrv-sensors` if needed
- `External options` → Enable `majestic` streamer

### Step 4: Modify Sensor Loading Script

Before building, modify the sensor loading script:

```bash
vi general/package/sigmastar-osdrv-infinity6/files/script/load_sigmastar
```

Add your sensor to the `set_sensor()` function as described in Step 4 above.

### Step 5: Build Firmware

```bash
make
```

The build process will take 30-60 minutes depending on your system.

Output files will be in: `output/images/`

### Step 6: Flash Firmware

```bash
# Copy rootfs and kernel to your device
scp output/images/rootfs.squashfs.* root@device-ip:/tmp/
scp output/images/uImage.* root@device-ip:/tmp/

# On the device, flash the firmware
sysupgrade --kernel=/tmp/uImage.* --rootfs=/tmp/rootfs.squashfs.* -z
```

## Troubleshooting

### Sensor Not Detected

**Symptoms:** `ipcinfo -s` returns empty or error

**Solutions:**
1. Check hardware connections (MIPI lanes, I2C, power)
2. Verify I2C communication:
   ```bash
   i2cdetect -y 0  # Try different bus numbers: 0, 1, 2
   ```
3. Check sensor power supply voltage
4. Verify reset/power-down GPIO states
5. Check kernel logs:
   ```bash
   dmesg | grep -i "sensor\|i2c\|mipi"
   ```

### Driver Loading Fails

**Symptoms:** `insmod` returns error

**Solutions:**
1. Check kernel version compatibility:
   ```bash
   uname -r
   ls /lib/modules/
   ```
2. Verify driver file exists and has correct permissions
3. Check dependencies:
   ```bash
   modprobe -D sensor_your_sensor_mipi
   ```
4. Review driver requirements (some need specific kernel configs)

### No Video Stream

**Symptoms:** Majestic runs but no video output

**Solutions:**
1. Check sensor binary configuration exists:
   ```bash
   ls -la /usr/bin/*.bin
   ```
2. Verify ISP settings in `/etc/majestic.yaml`
3. Check Majestic logs:
   ```bash
   logread | grep majestic
   ```
4. Verify resolution matches sensor capabilities
5. Test with lower resolution/framerate

### Image Quality Issues

**Symptoms:** Distorted, noisy, or incorrect colors

**Solutions:**
1. Adjust ISP parameters in `/etc/majestic.yaml`
2. Verify sensor configuration binary is correct for your sensor model
3. Check MCLK frequency matches sensor requirements
4. Verify MIPI lane configuration (chmap parameter)
5. Update sensor firmware/configuration

### I2C Communication Errors

**Symptoms:** I2C errors in dmesg, sensor not responding

**Solutions:**
1. Verify I2C bus number (try i2c-0, i2c-1, i2c-2)
2. Check I2C address (7-bit vs 8-bit addressing)
3. Reduce I2C clock speed (100kHz instead of 400kHz)
4. Check for I2C pull-up resistors (typically 4.7kΩ)
5. Verify I2C signals with logic analyzer

## Advanced Configuration

### MIPI Lane Configuration

Some sensors support different lane configurations:

```bash
# Single lane (chmap=1)
insmod sensor_sensor_name_mipi.ko chmap=1

# Dual lane (chmap=3)  
insmod sensor_sensor_name_mipi.ko chmap=3

# Quad lane (chmap=15)
insmod sensor_sensor_name_mipi.ko chmap=15
```

The `chmap` parameter is a bitmask:
- Bit 0: Lane 0
- Bit 1: Lane 1  
- Bit 2: Lane 2
- Bit 3: Lane 3

### Custom Sensor Configuration

For advanced users, sensor configuration binaries can be customized:

1. **Extract existing configuration:**
   ```bash
   strings /usr/bin/imx307.bin > sensor_config.txt
   ```

2. **Modify parameters** (exposure, gain, resolution, etc.)

3. **Rebuild binary** using sensor configuration tools

### Parallel Interface Sensors

Some sensors (like SC2239, SC2335) can work in parallel mode:

```bash
# Check SoC variant
if [ "$(fw_printenv -n soc)" = "ssc325de" ]; then
    IFACE=parl  # Use parallel interface
else
    IFACE=mipi  # Use MIPI interface
fi

insmod sensor_${SENSOR}_${IFACE}.ko
```

## Reference Files

### Key Files and Locations

| File | Purpose | Location |
|------|---------|----------|
| Sensor driver | Kernel module | `/lib/modules/4.9.84/sigmastar/sensor_*.ko` |
| Loading script | Initialize sensor | `/usr/sbin/load_sigmastar` |
| Sensor config | Binary configuration | `/usr/bin/*.bin` |
| Majestic config | Streamer settings | `/etc/majestic.yaml` |
| U-Boot env | Persistent settings | `/dev/mtd1` (accessed via `fw_printenv`/`fw_setenv`) |
| Init script | Vendor module loading | `/etc/init.d/S70vendor` |

### Source Code Locations (in firmware repo)

| Component | Path |
|-----------|------|
| Sensor loading script | `general/package/sigmastar-osdrv-infinity6/files/script/load_sigmastar` |
| Kernel modules | `general/package/sigmastar-osdrv-infinity6/files/kmod/` |
| Sensor configs | `general/package/sigmastar-osdrv-infinity6/files/sensor/configs/` |
| Majestic package | `general/package/majestic/` |
| Board configs | `br-ext-chip-sigmastar/board/infinity6/` |

## Additional Resources

- [OpenIPC Wiki](https://github.com/openipc/wiki) - General documentation
- [OpenIPC Firmware Repository](https://github.com/openipc/firmware) - Main firmware repo
- [OpenIPC Sensors Repository](https://github.com/openipc/sensors) - Sensor drivers
- [OpenIPC Telegram Chat](https://t.me/openipc) - Community support
- [Majestic Documentation](https://github.com/openipc/wiki/blob/master/en/majestic-streamer.md) - Streamer configuration

## Safety and Warranty

⚠️ **WARNING**: Modifying hardware and firmware voids warranty and may damage your device.

- Always backup original firmware before modifications
- Use proper ESD protection when handling electronics
- Double-check power supply voltages before connecting sensors
- Test on a spare device if possible
- Keep serial console access available for recovery

## Contributing

Found a new sensor configuration or improvement? Contribute to OpenIPC:

1. Test thoroughly on your hardware
2. Document your changes
3. Submit a pull request to the [firmware repository](https://github.com/openipc/firmware)
4. Share your findings in the [Telegram community](https://t.me/openipc)

## License

This guide is provided as-is for educational purposes. OpenIPC firmware is licensed under MIT License. Individual sensor drivers may have different licenses - check before using commercially.

---

**Last Updated:** 2025-12-06  
**Firmware Version:** OpenIPC 2024+  
**Target Platform:** RunCam WiFiLink2 (SigmaStar Infinity6)
