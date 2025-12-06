# Example: Adding IMX415 Sensor Support to RunCam WiFiLink2

This is a practical example showing exactly what needs to be changed to add support for a new MIPI sensor (IMX415) to the RunCam WiFiLink2 with SigmaStar Infinity6 SoC.

## Scenario

- **Device:** RunCam WiFiLink2 (SigmaStar SSC009B)
- **Original Sensor:** GC2053 (2MP)
- **New Sensor:** Sony IMX415 (8MP MIPI)
- **Goal:** Replace hardware sensor and add software support

## Hardware Changes

### 1. Physical Connection

Connect IMX415 module to the MIPI CSI interface:

| IMX415 Pin | SSC009B Signal | Notes |
|------------|----------------|-------|
| MIPI_CLK_P | MIPI_CLK_P | Differential pair |
| MIPI_CLK_N | MIPI_CLK_N | Keep traces matched |
| MIPI_D0_P | MIPI_DATA0_P | Data lane 0 |
| MIPI_D0_N | MIPI_DATA0_N | Required |
| MIPI_D1_P | MIPI_DATA1_P | Data lane 1 |
| MIPI_D1_N | MIPI_DATA1_N | For 8MP, both lanes needed |
| I2C_SDA | I2C1_SDA | Control interface |
| I2C_SCL | I2C1_SCL | Control interface |
| MCLK | MCLK_OUT | 27MHz or 24MHz |
| RESET | GPIO60 | Active low |
| VDD_IO | 1.8V | IO voltage |
| VDD_ANALOG | 2.8V | Analog voltage |
| VDD_CORE | 1.2V | Core voltage |
| GND | GND | Multiple connections |

**Important:** IMX415 requires 3 voltage rails. Verify your WiFiLink2 board has appropriate regulators.

## Software Changes

### Change 1: Modify Sensor Loading Script

**File:** `general/package/sigmastar-osdrv-infinity6/files/script/load_sigmastar`

**Before:**
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
		*)
			echo -e "\n\e[1;31mUNSUPPORTED sensor - $SENSOR\e[0m\n" | logger -s -t OpenIPC
			;;
	esac
}
```

**After:**
```bash
set_sensor() {
	case $SENSOR in
		gc2053|imx307|sc3335)
			insmod $MODULE/sensor_${SENSOR}_mipi.ko chmap=1
			;;
		imx415)
			# IMX415 requires 2 lanes for 8MP
			insmod $MODULE/sensor_${SENSOR}_mipi.ko chmap=3
			;;
		sc2239|sc2335)
			[ "$(fw_printenv -n soc)" = "ssc325de" ] && IFACE=parl
			insmod $MODULE/sensor_${SENSOR}_${IFACE:-mipi}.ko chmap=1
			;;
		*)
			echo -e "\n\e[1;31mUNSUPPORTED sensor - $SENSOR\e[0m\n" | logger -s -t OpenIPC
			;;
	esac
}
```

**Change Summary:**
- Added `imx415` case
- Used `chmap=3` for dual MIPI lanes (lanes 0 and 1)

### Change 2: Add Sensor Driver to Package

**File:** `general/package/sigmastar-osdrv-infinity6/sigmastar-osdrv-infinity6.mk`

If building from source, ensure sensor driver is included. Typically the driver comes from the sensors repository.

**Add driver file:**
- Source: Get `sensor_imx415_mipi.ko` from OpenIPC sensors repo or vendor
- Destination: `/lib/modules/4.9.84/sigmastar/sensor_imx415_mipi.ko`

### Change 3: Add Sensor Configuration

**Add configuration binary:**
- Source: Get `imx415.bin` from OpenIPC sensors repo or vendor
- Destination: `/usr/bin/imx415.bin`

This binary contains ISP calibration, AWB settings, etc.

### Change 4: Configure U-Boot Environment

On the device, set the sensor:

```bash
fw_setenv sensor imx415
```

Verify:
```bash
fw_printenv sensor
# Output: sensor=imx415
```

### Change 5: Update Majestic Configuration

**File:** `/etc/majestic.yaml`

**Before (for GC2053):**
```yaml
system:
  sensor: gc2053

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

**After (for IMX415):**
```yaml
system:
  sensor: imx415

image:
  width: 3840
  height: 2160

video0:
  codec: h265  # H.265 recommended for 4K
  width: 3840
  height: 2160
  fps: 20  # Lower FPS for 4K
  bitrate: 8192  # Higher bitrate for 4K

video1:
  enabled: true
  codec: h264
  width: 1920
  height: 1080
  fps: 30
  bitrate: 2048
```

**Changes:**
- Updated sensor name
- Changed resolution to 4K (3840x2160)
- Changed codec to H.265 for better compression
- Reduced FPS to 20 (4K is demanding)
- Increased bitrate
- Added secondary 1080p stream

### Change 6: GPIO Configuration (if needed)

If IMX415 needs specific GPIO control for reset/power:

**Create:** `/etc/init.d/S36sensor-gpio`

```bash
#!/bin/sh

case "$1" in
	start)
		echo "Configuring IMX415 GPIOs..."
		
		# Reset GPIO 60
		echo 60 > /sys/class/gpio/export 2>/dev/null
		echo out > /sys/class/gpio/gpio60/direction
		
		# Power down GPIO 61 (if needed)
		echo 61 > /sys/class/gpio/export 2>/dev/null
		echo out > /sys/class/gpio/gpio61/direction
		
		# Power sequence
		echo 1 > /sys/class/gpio/gpio61/value  # Disable power down
		usleep 5000
		echo 0 > /sys/class/gpio/gpio60/value  # Assert reset
		usleep 10000
		echo 1 > /sys/class/gpio/gpio60/value  # Release reset
		usleep 20000
		;;
		
	stop)
		echo 60 > /sys/class/gpio/unexport 2>/dev/null
		echo 61 > /sys/class/gpio/unexport 2>/dev/null
		;;
		
	*)
		echo "Usage: $0 {start|stop}"
		exit 1
		;;
esac
```

Make it executable:
```bash
chmod +x /etc/init.d/S36sensor-gpio
```

## Testing Steps

### Step 1: Verify Hardware

```bash
# Check I2C detection (IMX415 is usually at 0x1a)
i2cdetect -y 1

# Expected output:
#      0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
# 00:          -- -- -- -- -- -- -- -- -- -- -- -- --
# 10: -- -- -- -- -- -- -- -- -- -- 1a -- -- -- -- --
```

### Step 2: Load Sensor Driver

```bash
# Manual test
modprobe sensor_imx415_mipi chmap=3

# Check if loaded
lsmod | grep imx415

# Check kernel messages
dmesg | tail -20
```

Expected output:
```
[   10.123] sensor_imx415_mipi: loading
[   10.456] IMX415 sensor initialized
[   10.789] MIPI lanes: 2
```

### Step 3: Test Sensor Detection

```bash
# Use ipcinfo to detect sensor
ipcinfo -s
```

Expected output:
```
imx415
```

### Step 4: Test Video Stream

```bash
# Restart Majestic
/etc/init.d/S95majestic restart

# Check Majestic logs
logread | grep majestic

# Test RTSP stream
ffplay rtsp://192.168.1.10:554/stream=0
```

### Step 5: Verify Video Quality

Expected behavior:
- Image appears correctly oriented
- Colors are natural (not purple/green tint)
- Focus is adjustable
- Exposure adjusts automatically
- No frame drops or corruption

## Build Process

To build complete firmware with IMX415 support:

```bash
# Clone repository
git clone https://github.com/openipc/firmware.git
cd firmware

# Modify load_sigmastar script as shown above
vi general/package/sigmastar-osdrv-infinity6/files/script/load_sigmastar

# Configure for Infinity6 SSC009B
make infinity6-ssc009b_lite_defconfig

# Optional: Enable sensors package
make menuconfig
# Navigate to: External options → sigmastar-osdrv-sensors

# Build
make -j$(nproc)

# Output files
ls -lh output/images/
# openipc-infinity6-ssc009b-lite.tgz
# rootfs.squashfs.*
# uImage.*
```

## Flashing Firmware

```bash
# Copy to device
scp output/images/openipc-infinity6-ssc009b-lite.tgz root@192.168.1.10:/tmp/

# On device, extract and flash
cd /tmp
tar -xzf openipc-infinity6-ssc009b-lite.tgz
sysupgrade --kernel=uImage.infinity6-ssc009b --rootfs=rootfs.squashfs.infinity6-ssc009b -z

# Device will reboot automatically
```

## Common Issues and Solutions

### Issue 1: Sensor Not Detected

**Symptom:** `ipcinfo -s` returns empty

**Solution:**
```bash
# Check I2C bus
i2cdetect -y 0
i2cdetect -y 1

# Check GPIO states
cat /sys/kernel/debug/gpio

# Verify power supplies with multimeter
# AVDD: 2.8V
# DVDD: 1.2V
# IOVDD: 1.8V
```

### Issue 2: Purple/Green Image

**Symptom:** Video stream shows incorrect colors

**Solution:**
- Update `imx415.bin` configuration file
- Verify MCLK frequency (should be 27MHz or 24MHz)
- Check ISP settings in majestic.yaml

### Issue 3: No Video Output

**Symptom:** Majestic starts but no video

**Solution:**
```bash
# Check sensor driver loaded
lsmod | grep imx415

# Check MIPI lane configuration
dmesg | grep -i mipi

# Try lower resolution first
vi /etc/majestic.yaml
# Set to 1920x1080 for testing
```

### Issue 4: Frame Drops

**Symptom:** Video stutters or drops frames

**Solution:**
- Reduce FPS in majestic.yaml
- Lower bitrate
- Use H.265 instead of H.264
- Reduce resolution

## Performance Optimization

### For 4K Video:

```yaml
# /etc/majestic.yaml
video0:
  codec: h265
  width: 3840
  height: 2160
  fps: 15  # Lower FPS
  bitrate: 6144
  gopSize: 45  # 3 seconds at 15 fps
```

### For Lower Latency:

```yaml
video0:
  codec: h264
  width: 1920
  height: 1080
  fps: 30
  bitrate: 2048
  gopSize: 30  # 1 second
  profile: baseline  # Lower latency
```

## Summary of Files Changed

| File | Change Type | Description |
|------|-------------|-------------|
| `general/package/sigmastar-osdrv-infinity6/files/script/load_sigmastar` | Modified | Added IMX415 case with chmap=3 |
| `/etc/init.d/S36sensor-gpio` | Created | GPIO configuration for IMX415 |
| `/etc/majestic.yaml` | Modified | Updated for 4K resolution and H.265 |
| U-Boot env variable `sensor` | Set | Changed from `gc2053` to `imx415` |

## Driver Files Needed

| File | Location | Source |
|------|----------|--------|
| `sensor_imx415_mipi.ko` | `/lib/modules/4.9.84/sigmastar/` | OpenIPC sensors repo |
| `imx415.bin` | `/usr/bin/` | OpenIPC sensors repo |

## Complete Boot Sequence

1. U-Boot loads kernel
2. Kernel starts init system
3. `/etc/init.d/S35modules` loads base kernel modules
4. `/etc/init.d/S36sensor-gpio` configures GPIOs (our new script)
5. `/etc/init.d/S70vendor` runs `load_sigmastar -i`
6. `load_sigmastar` script:
   - Reads `sensor=imx415` from U-Boot env
   - Loads base SigmaStar modules (mi_sys, mi_vif, etc.)
   - Loads `sensor_imx415_mipi.ko chmap=3`
   - Sensor initialized
7. `/etc/init.d/S95majestic` starts Majestic streamer
8. Majestic reads `/etc/majestic.yaml` and `/usr/bin/imx415.bin`
9. Video stream becomes available

## Verification Checklist

- [ ] Hardware connected properly (MIPI, I2C, MCLK, power)
- [ ] I2C device detected at correct address
- [ ] Sensor driver added to `/lib/modules/4.9.84/sigmastar/`
- [ ] Sensor config binary added to `/usr/bin/`
- [ ] `load_sigmastar` script updated with new sensor
- [ ] U-Boot env variable set: `fw_setenv sensor imx415`
- [ ] GPIO script created and executable
- [ ] `majestic.yaml` configured for sensor resolution
- [ ] Device reboots successfully
- [ ] `ipcinfo -s` shows `imx415`
- [ ] `lsmod` shows `sensor_imx415_mipi`
- [ ] RTSP stream works and shows correct image

---

This example demonstrates the complete process of adding a new sensor. Apply the same pattern for other sensors by adjusting the sensor name, resolution, and specific parameters.
