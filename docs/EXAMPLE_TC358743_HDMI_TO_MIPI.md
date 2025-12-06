# HDMI to MIPI CSI-2 Adapter Integration Guide (TC358743XBG)

## Overview

This guide demonstrates how to integrate an HDMI to MIPI CSI-2 adapter based on the **Toshiba TC358743XBG** chip with RunCam WiFiLink2 or other OpenIPC-supported devices. This allows you to use any HDMI video source (camera, computer, game console, etc.) as input instead of a traditional camera sensor.

## What is TC358743XBG?

The TC358743XBG is an HDMI to MIPI CSI-2 bridge chip that:
- Converts HDMI 1.4 input to MIPI CSI-2 output
- Supports resolutions up to 1080p60
- Handles HDCP (though typically disabled for DIY projects)
- Provides I2C configuration interface
- Outputs up to 4 MIPI data lanes

## Hardware Requirements

### Components Needed

1. **TC358743XBG-based HDMI to MIPI adapter board**
   - Common modules: Waveshare HDMI to MIPI adapter, Auvidea B101/B102
   - Ensure it's designed for MIPI CSI-2 output (not DSI)

2. **RunCam WiFiLink2** or compatible OpenIPC device with:
   - SigmaStar Infinity6 SoC or Novatek NT9856X SoC
   - Available MIPI CSI-2 interface
   - Sufficient I2C bus
   - GPIO pins for control

3. **HDMI Video Source**
   - Camera with HDMI output
   - Computer/laptop
   - Game console
   - Media player
   - Any device with HDMI 1.4 output

4. **Tools**
   - Soldering equipment
   - Multimeter
   - Logic analyzer (optional, for debugging)
   - USB-TTL serial adapter

## Hardware Integration

### TC358743XBG Pin Configuration

#### Power Connections

| Pin | Function | Voltage | Notes |
|-----|----------|---------|-------|
| VDD_IO | I/O Power | 1.8V | Digital I/O supply |
| VDD_CORE | Core Power | 1.2V | Core logic supply |
| VDD_PLL | PLL Power | 1.2V | PLL power supply |
| VDD_MIPI | MIPI Power | 1.2V | MIPI transmitter supply |
| AVDD33 | Analog 3.3V | 3.3V | Analog circuits |

**Important:** The TC358743XBG requires multiple voltage rails. Most pre-built modules handle this internally.

#### MIPI CSI-2 Output

| Signal | Function | Connection | Notes |
|--------|----------|------------|-------|
| MIPI_CLK_P | Clock + | SoC MIPI_CLK_P | Differential pair |
| MIPI_CLK_N | Clock - | SoC MIPI_CLK_N | High-speed clock |
| MIPI_D0_P | Data 0 + | SoC MIPI_DATA0_P | Required |
| MIPI_D0_N | Data 0 - | SoC MIPI_DATA0_N | Required |
| MIPI_D1_P | Data 1 + | SoC MIPI_DATA1_P | Required for 1080p |
| MIPI_D1_N | Data 1 - | SoC MIPI_DATA1_N | Required for 1080p |
| MIPI_D2_P | Data 2 + | SoC MIPI_DATA2_P | Optional |
| MIPI_D2_N | Data 2 - | SoC MIPI_DATA2_N | Optional |
| MIPI_D3_P | Data 3 + | SoC MIPI_DATA3_P | Optional |
| MIPI_D3_N | Data 3 - | SoC MIPI_DATA3_N | Optional |

**For 1080p60:** Use at least 2 lanes (D0, D1)  
**For 4K:** Use 4 lanes (D0, D1, D2, D3) - not supported on most WiFiLink2 devices

#### I2C Control Interface

| Signal | Function | Connection | Default |
|--------|----------|------------|---------|
| SDA | I2C Data | SoC I2C_SDA | Pull-up to 1.8V |
| SCL | I2C Clock | SoC I2C_SCL | Pull-up to 1.8V |
| I2C Address | - | - | 0x0F (7-bit) |

**Important:** TC358743 I2C address is typically 0x0F (0x1E in 8-bit addressing).

#### Control Signals

| Signal | Function | Connection | Notes |
|--------|----------|------------|-------|
| RESET# | Reset | GPIO | Active low, pull high to operate |
| INT1 | Interrupt | GPIO (optional) | For HDMI hotplug detection |
| CSI_EN | MIPI Enable | Pull high | Or connect to GPIO |

### Typical Connection Diagram

```
HDMI Source  →  TC358743XBG Module  →  RunCam WiFiLink2 (SigmaStar SSC009)
                                     
HDMI Cable          MIPI CSI-2:
                    - CLK (differential)
                    - DATA0 (differential)
                    - DATA1 (differential)
                    
                    I2C:
                    - SDA → I2C1_SDA
                    - SCL → I2C1_SCL
                    
                    Control:
                    - RESET# → GPIO60
                    - INT1 → GPIO61 (optional)
                    - CSI_EN → 1.8V (always on)
                    
                    Power:
                    - 1.8V, 1.2V, 3.3V from module regulators
```

### Physical Connection Tips

1. **Keep MIPI traces short** - Less than 10cm if possible
2. **Match differential pair impedance** - 100Ω for MIPI
3. **Add AC coupling capacitors** - 100nF on MIPI lines if needed
4. **Pull-up resistors on I2C** - 4.7kΩ to 1.8V
5. **RESET pull-up** - 10kΩ to 1.8V
6. **Power sequence** - Power up 1.2V → 1.8V → 3.3V
7. **Decoupling capacitors** - 100nF close to power pins

## Platform-Specific Integration

### Option A: Novatek NT9856X (Native Support)

The TC358743 is already supported on Novatek platform.

#### 1. Enable TC358743 in Buildroot

The driver is included in the Novatek package at:
```
general/package/novatek-osdrv-nt9856x/files/sensor/sen_ad_tc358743/nvt_sen_ad_tc358743.ko
```

#### 2. Configure Sensor in Load Script

**File:** `general/package/novatek-osdrv-nt9856x/files/script/load_novatek`

Add TC358743 to the sensor loading logic:

```bash
insert_sns() {
    case $SENSOR in
    tc358743)
        # TC358743 HDMI to MIPI bridge configuration
        # Configure MIPI receiver for 2-lane mode
        devmem 0xf0220000 32 0x00001473
        devmem 0xF0220004 32 0x10000000
        devmem 0xF0220008 32 0x00048100
        devmem 0xF022000C 32 0x0034003e
        devmem 0xF0220010 32 0x30123860
        # ... additional register configuration
        
        insmod nvt_sen_ad_tc358743.ko
        ;;
    sc401ai | sc500ai | sc501ai)
        # ... existing sensors
        ;;
    *)
        echo "xxxx Invalid sensor type $SENSOR xxxx"
        ;;
    esac
}
```

#### 3. Set Sensor in U-Boot

```bash
fw_setenv sensor tc358743
```

#### 4. Configure Video Resolution

Edit `/etc/majestic.yaml`:

```yaml
system:
  sensor: tc358743

image:
  width: 1920
  height: 1080

video0:
  codec: h264
  width: 1920
  height: 1080
  fps: 60  # Depends on HDMI input
  bitrate: 4096
```

### Option B: SigmaStar Infinity6 (Custom Integration)

For RunCam WiFiLink2 with SigmaStar SoC, you need to add custom support.

#### 1. Create TC358743 Sensor Driver

Since SigmaStar doesn't have native TC358743 support, you have two options:

**Option B1: Port from Novatek**
- Extract driver from Novatek package
- Adapt to SigmaStar MI sensor interface
- This requires kernel driver development experience

**Option B2: Use V4L2 Subdev Driver (Recommended)**
- Linux kernel has mainline TC358743 driver
- Enable in kernel configuration
- More standard and maintainable

#### 2. Enable Kernel Driver

**File:** `br-ext-chip-sigmastar/board/infinity6/infinity6-ssc009a.config`

Add to kernel config:

```
CONFIG_VIDEO_TC358743=m
CONFIG_VIDEO_TC358743_CEC=y
CONFIG_MEDIA_SUPPORT=y
CONFIG_MEDIA_CAMERA_SUPPORT=y
CONFIG_V4L_PLATFORM_DRIVERS=y
```

#### 3. Create Device Tree Entry

Create or modify device tree file to add TC358743:

```dts
&i2c1 {
    tc358743: tc358743@0f {
        compatible = "toshiba,tc358743";
        reg = <0x0f>;
        
        clocks = <&mclk>;
        clock-names = "refclk";
        
        reset-gpios = <&gpio 60 GPIO_ACTIVE_LOW>;
        interrupt-parent = <&gpio>;
        interrupts = <61 IRQ_TYPE_LEVEL_HIGH>;
        
        port {
            tc358743_out: endpoint {
                remote-endpoint = <&mipi_csi_in>;
                data-lanes = <1 2>;
                clock-lanes = <0>;
                link-frequencies = /bits/ 64 <297000000>;
            };
        };
    };
};
```

#### 4. Modify Sensor Loading Script

**File:** `general/package/sigmastar-osdrv-infinity6/files/script/load_sigmastar`

```bash
set_sensor() {
    case $SENSOR in
        gc2053|imx307|sc3335)
            insmod $MODULE/sensor_${SENSOR}_mipi.ko chmap=1
            ;;
        tc358743)
            # Load V4L2 subdev driver
            modprobe tc358743
            # Configure MIPI for 2 lanes
            insmod $MODULE/mi_sensor.ko chmap=3
            # Additional configuration via media-ctl or sysfs
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

#### 5. GPIO Initialization Script

**File:** `/etc/init.d/S36tc358743-gpio`

```bash
#!/bin/sh

case "$1" in
    start)
        echo "Initializing TC358743 GPIOs..."
        
        # Reset pin (GPIO 60)
        echo 60 > /sys/class/gpio/export 2>/dev/null
        echo out > /sys/class/gpio/gpio60/direction
        
        # Interrupt pin (GPIO 61)
        echo 61 > /sys/class/gpio/export 2>/dev/null
        echo in > /sys/class/gpio/gpio61/direction
        
        # Reset sequence
        echo 0 > /sys/class/gpio/gpio60/value
        usleep 50000  # 50ms reset pulse
        echo 1 > /sys/class/gpio/gpio60/value
        usleep 100000  # 100ms startup time
        
        echo "TC358743 initialized"
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
chmod +x /etc/init.d/S36tc358743-gpio
```

## TC358743 Configuration

### I2C Register Configuration

The TC358743 requires extensive register configuration. Key registers:

```bash
# Example configuration script for TC358743
# I2C address: 0x0f

# Reset and initialization
i2cset -y 1 0x0f 0x0002 0x0001 w  # Software reset
usleep 100000

# HDMI PHY configuration
i2cset -y 1 0x0f 0x8520 0x0001 w  # HDMI PHY enable
i2cset -y 1 0x0f 0x8521 0x0000 w  # Auto input detection

# MIPI CSI-2 configuration
i2cset -y 1 0x0f 0x0140 0x0000 0004  # 2-lane mode
i2cset -y 1 0x0f 0x0144 0x0000 0000  # Continuous clock
i2cset -y 1 0x0f 0x0148 0x0000 0001  # Enable CSI output

# Video format (1080p60)
i2cset -y 1 0x0f 0x0006 0x0040 w  # FIFO level
i2cset -y 1 0x0f 0x0008 0x0000 w  # Video output format
```

**Note:** Full configuration is complex. Use existing configuration scripts or tools.

### Using media-ctl for Configuration

If using V4L2 driver:

```bash
# List media devices
media-ctl -d /dev/media0 -p

# Configure TC358743
media-ctl -d /dev/media0 --set-v4l2 '"tc358743 1-000f":0[fmt:UYVY8_2X8/1920x1080]'

# Configure link
media-ctl -d /dev/media0 -l '"tc358743 1-000f":0->"mipi-csi2":0[1]'
```

## Majestic Streamer Configuration

### For HDMI Input

**File:** `/etc/majestic.yaml`

```yaml
system:
  sensor: tc358743
  
isp:
  # HDMI input doesn't need ISP tuning
  # Disable or minimal ISP processing
  exposure: 0  # No exposure control
  gain: 0      # No gain control
  
image:
  width: 1920
  height: 1080
  
video0:
  enabled: true
  codec: h264
  width: 1920
  height: 1080
  fps: 60  # Match HDMI input
  gopSize: 60  # 1 second GOP
  bitrate: 8192
  profile: high
  
video1:
  enabled: true
  codec: h264
  width: 1280
  height: 720
  fps: 30
  bitrate: 2048
  
audio:
  enabled: true
  codec: aac
  sampleRate: 48000
  # TC358743 can extract HDMI audio
```

### Resolution Detection

Create a script to auto-detect HDMI input resolution:

**File:** `/usr/bin/detect-hdmi-resolution.sh`

```bash
#!/bin/sh

# Read TC358743 status registers via I2C
# This is a simplified example

I2C_BUS=1
I2C_ADDR=0x0f

# Read video timing registers
width=$(i2cget -y $I2C_BUS $I2C_ADDR 0x8573 w)
height=$(i2cget -y $I2C_BUS $I2C_ADDR 0x8575 w)
fps=$(i2cget -y $I2C_BUS $I2C_ADDR 0x8577 b)

echo "Detected: ${width}x${height}@${fps}"

# Update majestic configuration
# ... update /etc/majestic.yaml with detected values
```

## Testing and Verification

### Step 1: Hardware Verification

```bash
# Check I2C detection
i2cdetect -y 1

# Expected output:
#      0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
# 00:          -- -- -- -- -- -- -- -- -- -- -- -- --
# 10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 0f

# Read chip ID (should be 0x0000)
i2cget -y 1 0x0f 0x0000 w
```

### Step 2: Verify HDMI Input

```bash
# Check HDMI status registers
# Register 0x8520: HDMI detection status
status=$(i2cget -y 1 0x0f 0x8520 w)

if [ "$status" -eq "0x0001" ]; then
    echo "HDMI input detected"
else
    echo "No HDMI input"
fi
```

### Step 3: Check MIPI Output

```bash
# Verify driver is loaded
lsmod | grep tc358743

# Check kernel logs
dmesg | grep -i "tc358743\|mipi\|csi"

# Expected:
# tc358743 1-000f: tc358743 found @ 0x0f
# tc358743 1-000f: 1920x1080@60fps
```

### Step 4: Test Video Stream

```bash
# Restart Majestic
/etc/init.d/S95majestic restart

# Check logs
logread | grep majestic

# Test RTSP stream
ffplay rtsp://192.168.1.10:554/stream=0
```

## Common Issues and Solutions

### Issue 1: No I2C Communication

**Symptoms:** `i2cdetect` doesn't show device at 0x0f

**Solutions:**
1. Check I2C bus number (try 0, 1, 2)
2. Verify power supplies (1.2V, 1.8V, 3.3V)
3. Check I2C pull-up resistors (4.7kΩ to 1.8V)
4. Verify I2C level shifters if using 3.3V SoC
5. Try alternative I2C address (some modules use 0x0E)

### Issue 2: No HDMI Signal Detection

**Symptoms:** HDMI connected but no signal

**Solutions:**
1. Check HDMI cable quality
2. Verify HDMI source is outputting
3. Check TC358743 register 0x8520
4. Ensure RESET# is high (de-asserted)
5. Power cycle the TC358743 module
6. Try different HDMI resolution (start with 720p)

### Issue 3: No MIPI Output

**Symptoms:** HDMI detected but no video in Majestic

**Solutions:**
1. Check MIPI lane configuration (chmap)
2. Verify MIPI clock is running
3. Check CSI_EN signal is high
4. Verify MIPI differential signals with oscilloscope
5. Check for MIPI timing violations in dmesg
6. Reduce resolution to 720p30 for testing

### Issue 4: Corrupted Video

**Symptoms:** Video shows artifacts or wrong colors

**Solutions:**
1. Check MIPI lane count matches configuration
2. Verify pixel format (UYVY, RGB, etc.)
3. Adjust MIPI clock speed
4. Check for HDCP issues (disable if possible)
5. Verify continuous clock mode is enabled
6. Check FIFO settings in TC358743

### Issue 5: Audio Not Working

**Symptoms:** Video works but no audio

**Solutions:**
1. Enable I2S in TC358743 configuration
2. Check audio registers (0x8600-0x86FF)
3. Verify SoC has I2S input configured
4. Match sample rate (48kHz typical)
5. Check HDMI source is sending audio

## Advanced Configuration

### Multi-Format Support

Configure TC358743 to handle multiple input formats:

```bash
#!/bin/sh
# Auto-format detection script

detect_format() {
    # Read input timing
    h_active=$(i2cget -y 1 0x0f 0x8573 w)
    v_active=$(i2cget -y 1 0x0f 0x8575 w)
    
    # Configure based on detection
    if [ "$h_active" = "0x0780" ] && [ "$v_active" = "0x0438" ]; then
        echo "1920x1080 detected"
        # Configure for 1080p
    elif [ "$h_active" = "0x0500" ] && [ "$v_active" = "0x02D0" ]; then
        echo "1280x720 detected"
        # Configure for 720p
    fi
}
```

### EDID Programming

Program custom EDID to control HDMI source output:

```bash
# TC358743 has EDID RAM at 0x8C00-0x8CFF
# Program custom EDID for specific resolution

# Example: Force 1080p60
echo "Writing custom EDID..."
# ... i2cset commands to write EDID data
```

### Interrupt Handling

Use INT1 pin for hot-plug detection:

```bash
# Monitor GPIO for HDMI connect/disconnect
echo 61 > /sys/class/gpio/export
echo in > /sys/class/gpio/gpio61/direction
echo rising > /sys/class/gpio/edge

# Poll or use interrupt
cat /sys/class/gpio/gpio61/value
```

## Performance Optimization

### For Low Latency

```yaml
# majestic.yaml
video0:
  codec: h264
  gopSize: 15  # 0.5 second at 30fps
  profile: baseline
  slices: 4
```

### For High Quality

```yaml
video0:
  codec: h265
  bitrate: 12288  # 12 Mbps
  profile: main
  rcMode: cbr
```

## Use Cases

### 1. HDMI Camera Input

Replace traditional MIPI camera with HDMI camera:
- Professional cameras with HDMI out
- Action cameras (GoPro, DJI)
- Webcams with HDMI converter

### 2. Computer Screen Capture

Stream computer display via WiFiLink2:
- Remote desktop access
- Screen sharing
- Digital signage

### 3. Game Console Streaming

Stream gaming content:
- PlayStation, Xbox via HDMI
- Nintendo Switch
- Retro consoles with HDMI adapter

### 4. Video Switching

Multiple HDMI inputs with external switch:
- Multi-camera production
- Security monitoring from DVR
- Presentation systems

## Bill of Materials (BOM)

| Component | Example Part | Price | Notes |
|-----------|-------------|-------|-------|
| TC358743 Module | Waveshare HDMI-MIPI | $30-50 | Includes regulators |
| RunCam WiFiLink2 | Original or compatible | $40-60 | SigmaStar SoC preferred |
| HDMI Cable | Standard 1.4 | $5-10 | Keep short (<2m) |
| FFC Cable | 15-pin 0.5mm pitch | $2-5 | MIPI connection |
| Power Supply | 5V 2A | $5-10 | For complete system |
| **Total** | | **~$100** | Approximate |

## References

### Datasheets
- TC358743XBG Datasheet (Toshiba)
- SigmaStar SSC009B Datasheet
- MIPI CSI-2 Specification v1.3

### Software Resources
- Linux TC358743 driver: `drivers/media/i2c/tc358743.c`
- OpenIPC Wiki: https://github.com/openipc/wiki
- Device Tree bindings: `Documentation/devicetree/bindings/media/i2c/tc358743.txt`

### Tools
- `media-ctl` - Media controller configuration
- `v4l2-ctl` - V4L2 device control
- `i2c-tools` - I2C debugging

## Summary

The TC358743XBG HDMI to MIPI CSI-2 adapter enables:

✅ **Versatility** - Use any HDMI source as camera input  
✅ **Quality** - Support up to 1080p60 video  
✅ **Flexibility** - Change sources without hardware modification  
✅ **Audio** - Embed HDMI audio in stream  
✅ **Standards** - Based on MIPI CSI-2 standard interface  

**Platform Support:**
- ✅ **Novatek NT9856X** - Native driver support
- ⚠️ **SigmaStar Infinity6** - Requires custom driver integration
- ⚠️ **Other platforms** - Check for V4L2 support

**Key Advantages over Traditional Sensors:**
- No ISP tuning required
- Professional camera input support
- Easy source switching
- Computer/game console streaming

**Limitations:**
- More complex than standard sensor
- Requires multiple power rails
- HDCP may cause compatibility issues
- Latency slightly higher than direct sensor

---

**Last Updated:** 2025-12-06  
**Firmware Version:** OpenIPC 2024+  
**Target Platforms:** RunCam WiFiLink2 (SigmaStar), Novatek NT9856X  
**Example Chip:** Toshiba TC358743XBG
