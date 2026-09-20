# Experiment log

## Card identity

```text
GPU:        GeForce GTX 980M / GM204
PCI:        10DE:13D7
Subsystem:  1462:1129 (MSI)
Board ID:   E90C
VBIOS:      84.04.22.00.0C
EEPROM:     Macronix MX25L2005, 256 KB
```

Original saved ROM size: 185,344 bytes.

## Reference R1_EG

Reference file size: 199,680 bytes.

Observed reference identity included Board ID E906 and VBIOS version `84.04.22.00.0A`. Stock NVFlash therefore rejected it against the E90C card with a Board ID mismatch.

The reference image inserts a 14,336-byte EnableGop option-ROM component at file offset `0x1B200`, shifting the existing NVIDIA GOP. The underlying NVIDIA GOP following the insertion matched the original E90C GOP in the comparison performed during this project.

## E90C + EnableGop

Construction: E90C original as the base, with the exact EnableGop component from R1_EG inserted at `0x1B200` while retaining the original NVIDIA GOP.

Result:

```text
chime → no Apple logo → GOP/Zorin boot display → clean desktop
```

A post-flash readback matched the candidate hash exactly. This became the known-good recovery baseline.

## Full R1_EG with Board ID edited to E90C

Only the Board ID bytes at file offset `0x7B7` were changed from E906 (`06 e9`, little-endian) to E90C (`0c e9`).

Result:

```text
chime → white preboot screen → Zorin → artifacts
```

Conclusion: changing Board ID does not make the complete E906-oriented/reference payload safe for this E90C/MSI board.

## TEST1 — minimal DCB/connector transplant

Base: E90C + EnableGop.

Additional R1 ranges transplanted:

```text
0x5F8C–0x600B   DCB 4.1 device entries
0x612C–0x616B   connector entries
```

The E90C connector/platform byte was retained rather than importing the R1 platform change. No R1 power/clock/memory-related regions were intentionally transplanted.

Stock NVFlash 5.692 accepted the device/version identity but rejected the modified body with BIOS Cert 2.0 verification failure. A modified NVFlash 5.218 was used for the experimental write.

Result:

```text
chime → backlight → white screen → Zorin → Zorin splash → clean desktop
```

A Windows UEFI installer subsequently displayed successfully on the internal 21.5-inch panel.

## TEST2 — DCB/connector + BIT MXM metadata

Base: TEST1.

Additional bytes copied from R1:

```text
0x078E–0x0790
E90C: 30 01 2D
R1:   00 00 01
```

This area was interpreted during analysis as BIT MXM metadata associated with DCB modification status.

Result:

```text
chime → backlight → white screen → Zorin → Zorin splash → clean desktop
```

No observable improvement over TEST1. The Apple logo/native picker remained absent.

## Important negative result

The white preboot screen is not evidence that GOP is entirely absent. TEST1/TEST2 subsequently display Zorin boot graphics, and TEST2 displayed the Windows UEFI installer. The unresolved problem is specifically the earlier Apple preboot UI path.

## Remaining questions

- Can installed UEFI Windows boot directly from an internal disk with TEST2?
- Can OpenCore/BootKicker render or invoke the native Apple picker on this exact E90C/21.5-inch configuration?
- Is the white Apple preboot stage caused by a card-specific eDP initialization difference outside the DCB changes tested here?
