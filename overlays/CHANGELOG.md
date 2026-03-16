# Camera Overlay Changelog

## 2026-03-16 (v4): Switch VCM driver from dw9807 to ak7375

### Problem

The Arducam IMX519 B0371 autofocus cameras use an **AK7375** voice coil motor
(Asahi Kasei), not a DW9807 (Dongwoon). Both sit at I2C address 0x0c, so the
dw9807 driver bound successfully and I2C writes completed without error — but the
register protocol is completely different:

| | DW9807 | AK7375 |
|---|---|---|
| Position register | 0x03 (MSB) + 0x04 (LSB) | 0x00 (16-bit, val << 4) |
| Control register | 0x02, active=0x00, standby=0x01 | 0x02, active=0x00, standby=0x40 |
| DAC resolution | 10-bit (0–1023) | 12-bit (0–4095) |

The dw9807 DAC writes landed in the wrong registers with the wrong encoding, so
the lens motor barely moved. Focus sweep measurements (Laplacian variance on the
mug region) showed flat sharpness across all 1023 positions — confirmed by testing
a mug at 15cm and 25–30cm (well within the IMX519's 10cm minimum focus distance).

Source: [ArduCAM/IMX519_AK7375](https://github.com/ArduCAM/IMX519_AK7375) — the
official Arducam repo explicitly pairs the IMX519 with an ak7375 VCM driver.

### Fix

1. **New patched `ak7375.ko` module** built from the Arducam ak7375.c source, with
   the same CCI bus resilience patches applied in v3:
   - `ak7375_open()`: returns 0 (skip pm_runtime)
   - `ak7375_vcm_suspend/resume()`: tolerate I2C errors
   - `ak7375_probe()`: skip pm_runtime_idle
   - `ak7375_set_ctrl()`: lazy power-on (write REG_CONT=0x00 on first use)

2. **Updated both DT overlays**: `compatible` changed from `"dongwoon,dw9807-vcm"`
   to `"asahi-kasei,ak7375"` in both CSI1 and CSI2 overlay DTS files.

3. **Updated `/etc/modules-load.d/camera-vcm.conf`**: loads `ak7375` instead of
   `dw9807-vcm`.

The dw9807 patched module and source are retained for reference but are no longer
active.

### Files

| File | Description |
|------|-------------|
| `ak7375.c` | Patched ak7375 kernel module source |
| `ak7375-Makefile` | Out-of-tree build Makefile |
| `ak7375.ko.zst.ORIGINAL` | Stock kernel ak7375 module (for rollback) |
| `dw9807-vcm.c` | Previous patched dw9807 module (superseded) |
| `dw9807-vcm.ko.zst.ORIGINAL` | Stock kernel dw9807 module |

### Installation Locations

- `/lib/modules/6.8.0-1058-particle/kernel/drivers/media/i2c/ak7375.ko`
- `/mnt/userdata/boot/qcm6490-tachyon-camera-imx519-csi{1,2}.dtbo`
- `/boot/overlays/qcm6490-tachyon-camera-imx519-csi{1,2}.dtbo`
- `/etc/modules-load.d/camera-vcm.conf` → `ak7375`

### Rebuilding

```bash
cd /root/dev-projs/cameras/overlays
cp ak7375-Makefile Makefile
make
sudo cp ak7375.ko /lib/modules/$(uname -r)/kernel/drivers/media/i2c/
sudo depmod
# Also recompile overlays if DTS changed:
dtc -@ -I dts -O dtb -o qcm6490-tachyon-camera-imx519-csi1.dtbo qcm6490-tachyon-camera-imx519-csi1.dts
dtc -@ -I dts -O dtb -o qcm6490-tachyon-camera-imx519-csi2.dtbo qcm6490-tachyon-camera-imx519-csi2.dts
sudo mount /dev/sda10 /mnt/userdata
sudo cp qcm6490-tachyon-camera-imx519-csi*.dtbo /mnt/userdata/boot/
sudo cp qcm6490-tachyon-camera-imx519-csi*.dtbo /boot/overlays/
```

### Rollback to v3 (dw9807)

```bash
MODPATH="/lib/modules/$(uname -r)/kernel/drivers/media/i2c"
# Restore stock ak7375
sudo cp /root/dev-projs/cameras/overlays/ak7375.ko.zst.ORIGINAL "$MODPATH/ak7375.ko.zst"
sudo rm -f "$MODPATH/ak7375.ko"
# dw9807 patched module is still installed from v3
echo "dw9807-vcm" | sudo tee /etc/modules-load.d/camera-vcm.conf
# Revert overlays to dw9807-vcm compatible (would need to edit DTS and recompile)
sudo depmod && sudo reboot
```

### Usage

Focus range is now **0–4095** (4x resolution vs dw9807's 0–1023). Must still be
set during an active stream:
```bash
v4l2-ctl -d /dev/v4l-subdev30 --set-ctrl focus_absolute=2000  # CSI1
v4l2-ctl -d /dev/v4l-subdev28 --set-ctrl focus_absolute=2000  # CSI2
```

### Note on kernel upgrades

Both `ak7375.ko` and `dw9807-vcm.ko` are patched for `6.8.0-1058-particle`.
After a kernel upgrade, rebuild both from the sources in this directory.

### IMX519 Arducam B0371 optical specs (for reference)

- **Minimum focus distance**: 10cm (official), ~6cm (tested)
- **Sensor**: Sony IMX519, 1/2.53", 4656x3496, 1.22um pixels
- **FoV**: 66°(H) x 49.5°(V)
- **VCM**: AK7375 (Asahi Kasei), I2C address 0x0c, 12-bit DAC

---

## 2026-03-16 (v3): Patch dw9807-vcm kernel module for Qualcomm CCI bus (superseded by v4)

### Problem

The stock `dw9807-vcm` kernel module (v6.8.0-1058-particle) is incompatible with
Qualcomm's CCI (Camera Control Interface) bus. The CCI bus only powers on when the
camera subsystem is streaming, but the driver attempts I2C communication at three
points outside of streaming:

1. **`probe()`** calls `pm_runtime_idle()` which triggers immediate
   `dw9807_vcm_suspend()` → I2C write to CTL register → fails (`-110 ETIMEDOUT`
   on CCI0, `-6 ENXIO` on CCI1)
2. **`open()`** calls `pm_runtime_resume_and_get()` → `dw9807_vcm_resume()` →
   I2C write to CTL register → fails → returns `EINVAL` to userspace, making the
   V4L2 subdev unopenable
3. **`suspend()`** propagates I2C errors as fatal, putting the device into a PM
   error state that blocks all future opens

Additionally, the VCM's CTL register (0x02) was never written with the power-on
value (0x00), so even after making the subdev openable, the VCM motor stayed in
standby and `focus_absolute` writes had no physical effect.

The V4L2 subdev registered successfully (visible in media-ctl as `MEDIA_ENT_F_LENS`
with 0 pads), and `focus_absolute` controls were created, but the subdev could not
be opened — libcamera logged `Lens initialisation failed, lens disabled`.

### Fix

Rebuilt `dw9807-vcm.ko` from upstream v6.8 source with four targeted changes:

1. **`dw9807_open()`**: return 0 directly instead of calling
   `pm_runtime_resume_and_get()`. The CCI bus will be active when focus is
   actually set during streaming — no need to power up at open time.
2. **`dw9807_vcm_suspend()` / `dw9807_vcm_resume()`**: I2C failures log via
   `dev_dbg()` and return 0 instead of propagating errors. The CCI bus power
   lifecycle is managed externally by the camera subsystem, not by the VCM driver.
3. **`dw9807_probe()`**: removed `pm_runtime_idle()` call which triggered an
   immediate suspend with failing I2C.
4. **`dw9807_set_ctrl()`**: added lazy power-on — on first `focus_absolute` write,
   sends CTL=0x00 (power-on) to the VCM before setting the DAC position. A
   `powered_on` flag tracks state and resets in `suspend()` so the next stream
   session re-powers the VCM. This is necessary because `open()` no longer calls
   `pm_runtime_resume` (change 1), so the upstream resume path that wrote CTL=0x00
   is never reached.

### Verification

After patching, both VCMs register cleanly at boot, the V4L2 subdevs open
successfully, `focus_absolute` (range 0–1023) is exposed, and the VCM motor
physically responds:
```
v4l-subdev28: dw9807 17-000c  →  focus_absolute 0-1023
v4l-subdev30: dw9807 18-000c  →  focus_absolute 0-1023
```

dmesg confirms power-on on first use during streaming:
```
dw9807 17-000c: VCM powered on
dw9807 18-000c: VCM powered on
```

### Usage

Focus must be set **during an active camera stream** (CCI bus is only powered
while the camera subsystem is running):
```bash
# Set focus while a GStreamer/libcamera pipeline is running:
v4l2-ctl -d /dev/v4l-subdev30 --set-ctrl focus_absolute=500  # CSI1
v4l2-ctl -d /dev/v4l-subdev28 --set-ctrl focus_absolute=500  # CSI2
```

### Known issue: slow auto-exposure on first stream

The `ipa_soft_simple` software ISP starts auto-exposure from the sensor's current
exposure/gain values. After a fresh boot these default low, causing CSI2 in
particular to produce ~6 seconds of dark frames before converging.

**Workaround:** pre-seed exposure and gain on the sensor subdevs before streaming:
```bash
# CSI1 sensor = v4l-subdev29 (imx519 17-001a)
# CSI2 sensor = v4l-subdev27 (imx519 18-001a)
v4l2-ctl -d /dev/v4l-subdev29 --set-ctrl exposure=1000,analogue_gain=500
v4l2-ctl -d /dev/v4l-subdev27 --set-ctrl exposure=1000,analogue_gain=500
```

This brings frame 1 to proper brightness (488KB) vs the unassisted ramp (179KB at
frame 1, reaching 515KB only at frame ~380). The ISP's auto-exposure loop then
fine-tunes from this starting point.

**Why CSI1 doesn't always show the issue:** sensor controls persist across stream
sessions within the same boot. If CSI1 was streamed previously, its exposure/gain
are already high when the next stream starts. CSI2 shows it more reliably because
it's often tested second.

### Files

| File | Description |
|------|-------------|
| `dw9807-vcm.c` | Patched kernel module source |
| `dw9807-vcm-Makefile` | Out-of-tree build Makefile |
| `dw9807-vcm.ko.zst.ORIGINAL` | Stock kernel module (for rollback) |

### Installation Location

`/lib/modules/6.8.0-1058-particle/kernel/drivers/media/i2c/dw9807-vcm.ko`
(replaces the stock `.ko.zst`)

### Rebuilding

```bash
cd /root/dev-projs/cameras/overlays
cp dw9807-vcm-Makefile Makefile
make    # builds against running kernel headers
sudo cp dw9807-vcm.ko /lib/modules/$(uname -r)/kernel/drivers/media/i2c/
sudo depmod
```

### Rollback

```bash
MODPATH="/lib/modules/$(uname -r)/kernel/drivers/media/i2c"
sudo cp /root/dev-projs/cameras/overlays/dw9807-vcm.ko.zst.ORIGINAL \
        "$MODPATH/dw9807-vcm.ko.zst"
sudo rm -f "$MODPATH/dw9807-vcm.ko"
sudo depmod
sudo reboot
```

### Note on kernel upgrades

This patched module targets `6.8.0-1058-particle` specifically. After a kernel
package upgrade, the stock (broken) module will return. Re-run the build steps
above against the new kernel headers to re-apply the patch.

---

## 2026-03-16 (v2): Add VCM autofocus DT nodes to both overlays (superseded by v4)

### Problem

The firmware overlays had no VCM device tree node, so autofocus was non-functional.

### Fix

Added to both CSI1 and CSI2 overlays:
- `vcm@0c` node on the same I2C bus as the camera sensor
- `lens-focus = <&vcm_csiN>` phandle on the camera node linking sensor to VCM
- `/etc/modules-load.d/camera-vcm.conf` to autoload VCM module at boot

> **Note:** v2 originally used `compatible = "dongwoon,dw9807-vcm"`. v4 corrected
> this to `"asahi-kasei,ak7375"` after discovering the actual VCM chip.

Also created a proper CSI1 overlay DTS source (firmware only shipped compiled dtbo).

### Files

| File | Description |
|------|-------------|
| `qcm6490-tachyon-camera-imx519-csi1.dts` | CSI1 overlay source (with VCM) |
| `qcm6490-tachyon-camera-imx519-csi1.dtbo` | CSI1 compiled overlay |
| `qcm6490-tachyon-camera-imx519-csi1.dtbo.ORIGINAL` | Firmware CSI1 overlay (no VCM) |
| `qcm6490-tachyon-camera-imx519-csi2.dts` | CSI2 overlay source (with VCM) |
| `qcm6490-tachyon-camera-imx519-csi2.dtbo` | CSI2 compiled overlay |
| `qcm6490-tachyon-camera-imx519-csi2.dtbo.ORIGINAL` | Broken firmware CSI2 overlay |

### Rollback

To revert to pre-VCM overlays (cameras work, no autofocus):
```bash
sudo mount /dev/sda10 /mnt/userdata
sudo cp /root/dev-projs/cameras/overlays/qcm6490-tachyon-camera-imx519-csi1.dtbo.ORIGINAL \
        /mnt/userdata/boot/qcm6490-tachyon-camera-imx519-csi1.dtbo
# For CSI2, use the v1 fixed overlay (not the firmware ORIGINAL which was broken):
sudo cp /root/dev-projs/cameras/overlays/qcm6490-tachyon-camera-imx519-csi2.dtbo.ORIGINAL \
        /mnt/userdata/boot/qcm6490-tachyon-camera-imx519-csi2.dtbo
sudo rm /etc/modules-load.d/camera-vcm.conf
sudo reboot
```

---

## 2026-03-16 (v1): Fix imx519 CSI2 overlay (was broken in firmware)

### Problem

The shipped `qcm6490-tachyon-camera-imx519-csi2.dtbo` in Particle Tachyon Ubuntu
1.1.43 was an **exact byte-for-byte copy** of the CSI1 overlay. Both files had
md5 `ae1eb228f4bc9738758f868b196fb153`. This meant only one Arducam IMX519 camera
could ever be detected regardless of overlays.txt configuration.

The imx335 and imx219 overlays were correctly differentiated — only the imx519
pair had this bug.

### Root Cause

The CSI2 overlay targeted the same hardware as CSI1:
- CCI controller: `cci1` / `cci1_i2c0` (should be `cci0` / `cci0_i2c1`)
- CSIPHY port: port@3 / csiphy3 (should be port@1 / csiphy1)
- Reset GPIO: 141 (should be 21)
- Clock: MCLK3 / 0x62 (should be MCLK1 / 0x5e)
- Pinctrl: `csi1_*` (should be `csi2_*`)
- Missing: DSI/CSI switch pin (gpio68 output-high)

### Fix

Created a new `qcm6490-tachyon-camera-imx519-csi2.dtbo` modeled on the
known-good `qcm6490-tachyon-camera-imx335-csi2.dtbo` structure, adapted for
imx519 sensor properties (2 data lanes, 493.5 MHz link frequency, VANA/VDIG/VDDL
supply names).

### Files

| File | Description |
|------|-------------|
| `qcm6490-tachyon-camera-imx519-csi2.dts` | Source for the fixed overlay |
| `qcm6490-tachyon-camera-imx519-csi2.dtbo` | Compiled fixed overlay |
| `qcm6490-tachyon-camera-imx519-csi2.dtbo.ORIGINAL` | Broken overlay from firmware (for rollback) |

### Installation Locations

- `/mnt/userdata/boot/qcm6490-tachyon-camera-imx519-csi2.dtbo` (bootloader reads this)
- `/boot/overlays/qcm6490-tachyon-camera-imx519-csi2.dtbo` (system copy)

### Rollback

To revert to the original (broken) overlay:
```bash
sudo mount /dev/sda10 /mnt/userdata
sudo cp /root/dev-projs/cameras/overlays/qcm6490-tachyon-camera-imx519-csi2.dtbo.ORIGINAL \
        /mnt/userdata/boot/qcm6490-tachyon-camera-imx519-csi2.dtbo
sudo cp /root/dev-projs/cameras/overlays/qcm6490-tachyon-camera-imx519-csi2.dtbo.ORIGINAL \
        /boot/overlays/qcm6490-tachyon-camera-imx519-csi2.dtbo
sudo reboot
```

### Recompiling

If the DTS needs changes:
```bash
dtc -@ -I dts -O dtb \
    -o qcm6490-tachyon-camera-imx519-csi2.dtbo \
    qcm6490-tachyon-camera-imx519-csi2.dts
```

### Key Differences: CSI1 vs CSI2

| Property | CSI1 | CSI2 |
|----------|------|------|
| CCI controller | `cci1` (`cci@ac4b000`) | `cci0` (`cci@ac4a000`) |
| I2C bus | `cci1_i2c0` (i2c-bus@0) | `cci0_i2c1` (i2c-bus@1) |
| CSIPHY | port@3 (csiphy3) | port@1 (csiphy1) |
| Reset GPIO | 141 (0x8d) | 21 (0x15) |
| Clock ID | MCLK3 (0x62 / 98) | MCLK1 (0x5e / 94) |
| Pinctrl prefix | `csi1_*` | `csi2_*` |
| DSI/CSI switch | not needed | gpio68 output-high |
| Link frequency | 493,500,000 Hz | 493,500,000 Hz (same — per sensor, not per port) |
| Data lanes | 2 (0, 1) | 2 (0, 1) (same — per sensor) |

### Notes on Link Frequency

The `link-frequencies` property specifies the MIPI CSI-2 serial clock rate
generated by the sensor's internal PLL. It is a **sensor property**, not a port
property. Both CSI1 and CSI2 use the same IMX519 sensor, so both use 493.5 MHz.
This is confirmed by the imx335 overlays which also use identical link
frequencies (594 MHz) across CSI1 and CSI2.
