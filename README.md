# GTX 980M E90C in a 21.5-inch 2011 iMac (iMac12,1)

Experimental notes and recovery-oriented guide by **mrpulkkinen**.

> [!WARNING]
> This is not a generic GTX 980M flashing guide. The tested card is an MSI-subsystem GTX 980M with PCI ID `10DE:13D7`, subsystem `1462:1129`, Board ID `E90C`, and original VBIOS `84.04.22.00.0C`. Flashing a VBIOS can brick a GPU. Back up your own ROM first and have an external SPI recovery plan.

## Tested hardware

- Apple 21.5-inch 2011 iMac / iMac12,1
- NVIDIA GeForce GTX 980M (GM204)
- PCI ID: `10DE:13D7`
- Subsystem: `1462:1129` (MSI)
- Board ID: `E90C`
- Original VBIOS: `84.04.22.00.0C`
- EEPROM: Macronix MX25L2005, 2.7–3.6 V, 2048 Kbit / 256 KB
- Linux test environment: Zorin OS

## Goal

The original card booted Linux once the OS graphics stack took over, but did not provide useful Apple preboot graphics. The experiments attempted to add GOP/preboot support while preserving the E90C-specific board configuration.

## Confirmed results

| Firmware | Preboot behavior | Zorin | Notes |
| --- | --- | --- | --- |
| Original E90C | No useful EFI display | Clean | Baseline |
| E90C + EnableGop | GOP/Zorin boot graphics | Clean | Stable recovery baseline |
| Full R1_EG with only Board ID changed E906→E90C | White preboot | **Artifacts** | Do not use on this tested E90C card |
| TEST1: E90C + EnableGop + R1 DCB/connector entries | Backlight → white → Zorin | Clean | Windows UEFI installer also displayed |
| TEST2: TEST1 + R1 BIT MXM metadata | Backlight → white → Zorin | Clean | No visible improvement over TEST1 |

The Apple logo/native Apple picker has **not** been restored. However, the current experimental path successfully displayed the Windows UEFI installer on the internal 21.5-inch panel.

## Critical rule: make a backup first

With a compatible NVFlash build, save the installed firmware before changing anything:

```bash
sudo ./nvflash --save GTX980M_E90C_original.rom
sha256sum GTX980M_E90C_original.rom
```

The original ROM used in this experiment hashes to:

```text
fecaedfbafeed0a2f419e4c6138f4d2d410d25ec9a8bcf67ea835420c815d925
```

Your own card does **not** need to have this hash. If it differs, assume your firmware may be different and do not blindly use these experimental patches.

## Linux: unloading the NVIDIA driver

NVFlash refuses to write while the NVIDIA kernel driver is active. Switch to a text target and stop the display manager:

```bash
sudo systemctl isolate multi-user.target
sudo systemctl stop gdm3
sudo systemctl stop nvidia-persistenced 2>/dev/null
sudo systemctl stop nvidia-powerd 2>/dev/null

sudo rmmod nvidia_uvm
sudo rmmod nvidia_drm
sudo rmmod nvidia_modeset
sudo rmmod nvidia

lsmod | grep -E '^nvidia'
```

The final command should print nothing.

### If the modules immediately reload

Create a temporary hard block:

```bash
sudo systemctl set-default multi-user.target

sudo tee /etc/modprobe.d/nvidia-flash-blacklist.conf >/dev/null <<'BLOCK'
blacklist nvidia
blacklist nvidia_drm
blacklist nvidia_modeset
blacklist nvidia_uvm
install nvidia /bin/false
install nvidia_drm /bin/false
install nvidia_modeset /bin/false
install nvidia_uvm /bin/false
BLOCK

sudo update-initramfs -u -k all
sudo reboot
```

After reboot, verify `lsmod | grep -E '^nvidia'` is empty before using NVFlash.

After flashing/recovery, restore normal boot:

```bash
sudo rm -f /etc/modprobe.d/nvidia-flash-blacklist.conf
sudo update-initramfs -u -k all
sudo systemctl set-default graphical.target
sudo reboot
```

## NVFlash observations

Stock Linux NVFlash 5.692 recognized the card correctly as:

```text
GeForce GTX 980M (10DE,13D7,1462,1129)
```

Stock 5.692 rejected modified experimental firmware with:

```text
BIOS Cert 2.0 Verification Error, Update aborted.
ERROR: Invalid firmware image detected.
```

A modified NVFlash 5.218 identified as `Modified Version by Joe Dirt` was able to write the experimental images. This repository does not redistribute NVFlash. Obtain tools from their legitimate source and verify what you are running.

Before confirming any write, verify that NVFlash reports the expected IDs. For this tested card:

```text
Version: 84.04.22.00.0C
ID:      10DE:13D7:1462:1129
```

Do not proceed through unexpected board/device/subsystem mismatches merely because a patched flasher permits it.

## What was learned from the R1_EG experiment

The reference R1_EG firmware was intended to add iMac display-table changes plus EnableGop to GTX 980M cards. On this E90C/MSI card, transplanting the complete R1 firmware and changing only its Board ID from E906 to E90C produced a white preboot screen and artifacts when Zorin loaded. Reverting to the E90C-based firmware restored clean graphics.

Binary comparison showed that R1_EG contains more than EnableGop: it also changes legacy display/DCB data and other board/integrity-related regions. Therefore Board ID alone is not a sufficient compatibility test.

TEST1 deliberately retained the E90C base and added only EnableGop plus the R1 DCB/connector changes. TEST2 additionally changed the relevant BIT MXM metadata. Both remained clean in Linux but still showed a white Apple preboot stage rather than an Apple logo.

See [EXPERIMENTS.md](EXPERIMENTS.md) for the byte-level notes.

## Recovery

The safest software recovery image from this experiment was the E90C base with EnableGop. If an experimental ROM still allows Linux/SSH to boot, unload the NVIDIA modules and flash the known-good backup.

If the GPU no longer initializes sufficiently for software flashing, the physical fallback is an external SPI programmer connected to the MX25L2005 EEPROM. The EEPROM is a **2.7–3.6 V part**; do not apply 5 V to it.

## Windows

With TEST2 installed and no hard disk connected, a Windows UEFI installer USB produced visible Windows boot graphics and reached Windows Setup on the internal panel. This establishes that lack of an Apple logo does not mean UEFI Windows graphics are unusable.

The installed-to-disk Windows test is still pending and should be added here once confirmed.

## Files and hashes

See [SHA256SUMS.txt](SHA256SUMS.txt). Firmware files are intentionally not all redistributed here. A hash identifies a tested file; it does not establish compatibility with another GTX 980M.

## References / credit

This work builds on the iMac Maxwell/Pascal GPU-upgrade work by m0bil and other MacRumors contributors, Acidanthera/OpenCore EnableGop documentation, and NVIDIA's published DCB/BIT documentation. Add canonical source links in the GitHub repository description/References section when publishing.

## Status

**Experimental.** The practical result so far is clean Zorin graphics plus working Windows UEFI installer graphics on the internal panel. Native Apple-logo/Option-picker output remains unresolved.
