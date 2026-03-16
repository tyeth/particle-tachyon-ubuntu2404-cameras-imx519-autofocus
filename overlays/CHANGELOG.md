# Camera Overlay Changelog

## 2026-03-16 (v3): Patch dw9807-vcm kernel module for Qualcomm CCI bus

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

## 2026-03-16 (v2): Add dw9807 VCM autofocus support to both overlays

### Problem

The Arducam IMX519 autofocus cameras have a dw9807 voice coil motor (VCM) at I2C
address 0x0c on the same CCI bus as the sensor. The firmware overlays had no VCM
device tree node, so autofocus was non-functional — no `V4L2_CID_FOCUS_ABSOLUTE`
control was exposed, and the VCM driver couldn't probe because the Qualcomm CCI
bus only powers on during camera subsystem initialization (chicken-and-egg: driver
needs I2C access, but CCI only active when camera pipeline is up).

### Fix

Added to both CSI1 and CSI2 overlays:
- `vcm@0c` node with `compatible = "dongwoon,dw9807-vcm"` on the same I2C bus
  as the camera sensor
- `lens-focus = <&vcm_csiN>` phandle on the camera node linking sensor to VCM
- `/etc/modules-load.d/camera-vcm.conf` to autoload `dw9807-vcm` module at boot

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
