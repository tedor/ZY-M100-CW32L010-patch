# ZY-M100 CW32L010 Hardware Revision — Firmware Patch

## Background

The Tuya ZY-M100 24GHz mmWave presence sensor (model `TZE204_gkfbdvyx`) has a newer hardware revision that uses a **Wuhan Xinyuan CW32L010F8** MCU (ARM Cortex-M0+, 64KB Flash, 4KB RAM) instead of the GD32E230F8P6 (Cortex-M23) found in earlier revisions.

**WARNING:** Existing custom firmware `.bin` files from [Andrik45719/ZY-M100](https://github.com/Andrik45719/ZY-M100) and [gekkehenkie11/ZY-M100-Sensor-patching](https://github.com/gekkehenkie11/ZY-M100-Sensor-patching) are compiled for GD32E230 and **will brick this hardware revision** due to incompatible instruction sets (ARMv8-M vs ARMv6-M) and memory maps.

### Supported device

- **Board label:** `ZY-V250829-5.842G-V2.0CW`
- **Tuya ID:** `TZE204_gkfbdvyx`
- **MCU:** Wuhan Xinyuan CW32L010F8 (ARM Cortex-M0+, 64KB Flash)
- **Firmware version:** `2.0.3`, Product ID: `gkfbdvyx`

This hardware revision was first reported in [Andrik45719/ZY-M100#35](https://github.com/Andrik45719/ZY-M100/issues/35).

| ![Board top](https://github.com/user-attachments/assets/228f118f-c202-41b2-9c26-875b65450ded) | ![Board bottom](https://github.com/user-attachments/assets/5cae263d-5b14-448d-9482-4d8faaf0dfa1) | ![Board label](https://github.com/user-attachments/assets/e0525b3e-ec9b-4cac-8bed-61fc751e0d9e) |
|:---:|:---:|:---:|
| Board top | Board bottom | Board label |

| ![MCU closeup](https://github.com/user-attachments/assets/fb84ecc4-1d5c-40e0-a260-1285281eb209) | ![Radar module](https://github.com/user-attachments/assets/9e31d9c4-7d97-4017-b3fb-5d700193381d) | ![Device case](https://github.com/user-attachments/assets/29fa1c20-69a4-4c14-af08-e585dae8336f) |
|:---:|:---:|:---:|
| MCU closeup | Radar module | Device case |

*Photos by [@mime24](https://github.com/mime24) from [issue #35](https://github.com/Andrik45719/ZY-M100/issues/35)*

### How to identify your revision

- OpenOCD fails with `UNEXPECTED idcode: 0x0bc11477` when using `gd32e23x.cfg`
- pyOCD detects a **Cortex-M0+** core (instead of Cortex-M23)
- Dumped firmware contains strings like `cw32l010_adc.c`, `cw32l010_sysctrl.c`, `cw32l010_uart.c`

## Firmware variants

Three patched firmware files are provided, from least to most aggressive:

### `TZE204_gkfbdvyx_CW32L010_patched_td0_only.bin` — Minimal

Smallest change. Only disables the DP 9 = 0 spam.

| What | Effect |
|------|--------|
| Disable DP 9 `target_distance = 0` | Stops the 1-2 second flood of zero-distance reports when no target is detected. Actual distance reports are preserved. |

### `TZE204_gkfbdvyx_CW32L010_patched.bin` — Recommended

Disables DP 9 = 0 spam and reduces settings report frequency.

| What | Effect |
|------|--------|
| Disable DP 9 `target_distance = 0` | Same as above |
| Settings report interval 10s → 60s | DPs like sensitivity, detection range, etc. are re-sent every 60 seconds instead of every 10 seconds |

### `TZE204_gkfbdvyx_CW32L010_patched_no_dp104.bin` — Maximum silence

Everything from the recommended patch, plus silences DP 104 entirely.

| What | Effect |
|------|--------|
| Disable DP 9 `target_distance = 0` | Same as above |
| Settings report interval 10s → 60s | Same as above |
| Disable DP 104 (illuminance/lux) | Completely stops lux reports via DP 104. Use this if you don't need illuminance data in Home Assistant. |

### Comparison with GD32 patches

The GD32 firmware (patched by Andrik45719 and gekkehenkie11) also silences DP 255. This DP is **not present** in the CW32L010 firmware v2.0.3, so no equivalent patch is needed.

## Tools

OpenOCD does not have a native flash driver for CW32L010. Use **pyOCD** with the official CMSIS-Pack instead.

```bash
# NixOS
nix-shell -p python3Packages.pyocd

# pip
pip install pyocd
```

Download `WHXY.CW32L010_DFP.1.0.2.pack` from [the vendor](https://www.whxy.com) or CMSIS pack repositories. Place it in your working directory.

> **Note:** Cheap ST-Link V2 clones may need a firmware update to at least **V2J24** (via [STSW-LINK007](https://www.st.com/en/development-tools/stsw-link007.html)) for pyOCD to recognize them.

## Backup original firmware

**Always back up before flashing!**

```bash
sudo pyocd cmd \
  --pack ./WHXY.CW32L010_DFP.1.0.2.pack \
  -t cw32l010f8 \
  -c "savemem 0x00000000 0x10000 TZE204_gkfbdvyx_CW32L010.bin"
```

Verify the dump:
- File size should be exactly **65,536 bytes**
- Should contain the string `gkfbdvyx` (`strings TZE204_gkfbdvyx_CW32L010.bin | grep gkfbdvyx`)

## Flash patched firmware

Replace the filename with your chosen variant:

```bash
sudo pyocd flash \
  --pack ./WHXY.CW32L010_DFP.1.0.2.pack \
  -t cw32l010f8 \
  TZE204_gkfbdvyx_CW32L010_patched.bin \
  -a 0x00000000
```

## Restore original firmware

If something goes wrong, flash your backup:

```bash
sudo pyocd flash \
  --pack ./WHXY.CW32L010_DFP.1.0.2.pack \
  -t cw32l010f8 \
  TZE204_gkfbdvyx_CW32L010.bin \
  -a 0x00000000
```

## Applicability

These patches were developed for:
- **Model:** ZY-M100 24GHz mmWave presence sensor
- **Tuya ID:** `TZE204_gkfbdvyx`
- **MCU:** CW32L010F8 (Cortex-M0+)
- **Firmware version:** `2.0.3`
- **Product ID:** `gkfbdvyx`

If your firmware dump has a different version or product ID, the patched binaries may not be compatible with your device.

## Credits

- Patch logic derived from [Andrik45719/ZY-M100](https://github.com/Andrik45719/ZY-M100) and [gekkehenkie11/ZY-M100-Sensor-patching](https://github.com/gekkehenkie11/ZY-M100-Sensor-patching) (GD32 versions)
- CW32L010 adaptation: binary analysis of the CW32L010 firmware dump to locate equivalent code paths
