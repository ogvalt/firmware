# MIPI Camera Replacement - Quick Reference

## Quick Checklist

- [ ] Identify hardware: Run `ipcinfo -c` and `ipcinfo -s`
- [ ] Verify sensor compatibility with SigmaStar Infinity6
- [ ] Connect MIPI hardware (CLK, DATA lanes, I2C, MCLK, GPIOs)
- [ ] Add sensor driver to `/lib/modules/4.9.84/sigmastar/`
- [ ] Update sensor loading script `/usr/sbin/load_sigmastar`
- [ ] Set sensor in U-Boot: `fw_setenv sensor your_sensor`
- [ ] Configure Majestic in `/etc/majestic.yaml`
- [ ] Reboot and test

## Supported Sensors (Infinity6)

| Sensor | Type | Resolution | Interface |
|--------|------|------------|-----------|
| GC2053 | GalaxyCore | 2MP | MIPI |
| IMX307 | Sony | 2MP | MIPI |
| SC3335 | SmartSens | 3MP | MIPI |
| SC2239 | SmartSens | 2MP | MIPI/Parallel |
| SC2335 | SmartSens | 2MP | MIPI/Parallel |

## Essential Commands

```bash
# Hardware Info
ipcinfo -c          # Show SoC
ipcinfo -s          # Show sensor
ipcinfo -v          # Show vendor

# Sensor Configuration
fw_setenv sensor imx307              # Set specific sensor
fw_setenv sensor ""                  # Enable auto-detect
fw_printenv sensor                   # Check current setting

# Testing
dmesg | grep sensor                  # Check sensor detection
lsmod | grep sensor                  # Check loaded drivers
i2cdetect -y 0                       # Scan I2C bus
logread | grep majestic              # Check streamer logs

# Video Testing
curl http://device-ip/api/v1/video   # API check
rtsp://device-ip:554/stream=0        # RTSP stream
```

## Common MIPI Connections

```
Sensor          SoC
------          ---
MIPI_CLK+  -->  MIPI_CLK_P
MIPI_CLK-  -->  MIPI_CLK_N
MIPI_D0+   -->  MIPI_DATA0_P
MIPI_D0-   -->  MIPI_DATA0_N
MIPI_D1+   -->  MIPI_DATA1_P (optional)
MIPI_D1-   -->  MIPI_DATA1_N (optional)
SCL        -->  I2C_SCL
SDA        -->  I2C_SDA
MCLK       <--  MCLK (24/27MHz)
RESET      <--  GPIO
PWDN       <--  GPIO
VDD        <--  3.3V or 1.8V
GND        -->  GND
```

## Sensor Loading Script Template

File: `/usr/sbin/load_sigmastar` or in source: `general/package/sigmastar-osdrv-infinity6/files/script/load_sigmastar`

```bash
set_sensor() {
    case $SENSOR in
        gc2053|imx307|sc3335)
            insmod $MODULE/sensor_${SENSOR}_mipi.ko chmap=1
            ;;
        your_new_sensor)
            insmod $MODULE/sensor_${SENSOR}_mipi.ko chmap=1
            ;;
        *)
            echo -e "\n\e[1;31mUNSUPPORTED sensor - $SENSOR\e[0m\n" | logger -s -t OpenIPC
            ;;
    esac
}
```

## Majestic Configuration Template

File: `/etc/majestic.yaml`

```yaml
system:
  sensor: imx307

isp:
  slowShutter: true
  exposure: 50

image:
  width: 1920
  height: 1080

video0:
  codec: h264
  width: 1920
  height: 1080
  fps: 30
  bitrate: 2048
```

## Build Commands

```bash
# Clone firmware
git clone https://github.com/openipc/firmware.git
cd firmware

# Configure for Infinity6
make infinity6-ssc009b_lite_defconfig

# Optional: Configure packages
make menuconfig

# Build (30-60 minutes)
make

# Output location
ls output/images/
```

## MIPI Lane Configuration (chmap)

```bash
chmap=1   # Lane 0 only (0b0001)
chmap=3   # Lane 0+1    (0b0011)
chmap=15  # Lane 0+1+2+3 (0b1111)
```

## Troubleshooting Quick Fixes

| Problem | Quick Fix |
|---------|-----------|
| Sensor not detected | Check I2C with `i2cdetect -y 0` |
| Driver won't load | Verify kernel version matches driver |
| No video | Check `logread \| grep majestic` |
| Black image | Verify GPIO reset/power sequence |
| I2C errors | Check pull-up resistors (4.7kΩ) |
| Wrong colors | Update sensor .bin config file |

## File Locations Reference

| What | Runtime Path | Source Path |
|------|-------------|-------------|
| Sensor driver | `/lib/modules/4.9.84/sigmastar/sensor_*.ko` | Package files |
| Load script | `/usr/sbin/load_sigmastar` | `general/package/sigmastar-osdrv-infinity6/files/script/` |
| Sensor config | `/usr/bin/*.bin` | `general/package/sigmastar-osdrv-infinity6/files/sensor/configs/` |
| Majestic config | `/etc/majestic.yaml` | `general/package/majestic/files/` |
| Init script | `/etc/init.d/S70vendor` | `general/overlay/etc/init.d/` |

## GPIO Configuration Example

Create `/etc/init.d/S36sensor-gpio`:

```bash
#!/bin/sh

case "$1" in
    start)
        # Reset GPIO 60
        echo 60 > /sys/class/gpio/export
        echo out > /sys/class/gpio/gpio60/direction
        echo 0 > /sys/class/gpio/gpio60/value
        usleep 10000
        echo 1 > /sys/class/gpio/gpio60/value
        ;;
    stop)
        echo 60 > /sys/class/gpio/unexport
        ;;
esac
```

Make executable: `chmod +x /etc/init.d/S36sensor-gpio`

## Resources

- **Full Guide:** [MIPI_CAMERA_REPLACEMENT_GUIDE.md](MIPI_CAMERA_REPLACEMENT_GUIDE.md)
- **Wiki:** https://github.com/openipc/wiki
- **Firmware:** https://github.com/openipc/firmware
- **Sensors:** https://github.com/openipc/sensors
- **Support:** https://t.me/openipc
