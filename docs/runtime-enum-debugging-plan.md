# Runtime control-enumeration debugging plan

This change is evidence collection only. It does not alter the intended firmware protocol or attempt to fix probe failures.

## Instrumentation added

Both legacy OBISP driver copies now print the same control-enumeration log shape requested for comparing against the Raspberry Pi in-tree `arducam-pivariety` patch:

- `Discovering controls...` before count discovery.
- Every count-discovery `CTRL_INDEX_REG` write and `CTRL_ID_REG` read, including index, register name/address, value, and errno.
- `control[N] id=...` for every discovered control.
- `Detected N controls` after count discovery.
- Every descriptor transaction for each control index.
- `ENUM FAILED` before every `return ret` / `return -ENODEV` path inside control enumeration, with index, step, errno, and reason.

The Pivariety instrumentation patch in `patches/instrument-pivariety-enum-controls.patch` is the matching patch to apply to the Raspberry Pi kernel tree. Its distinguishing extra transactions are `CTRL_VALUE_REG <- 0` and polling `SYSTEM_IDLE_REG` after each descriptor index write.

## Direct comparison table

| Step | Legacy OBISP | Pivariety |
| --- | --- | --- |
| Print `Discovering controls...` | yes | yes |
| Count `CTRL_INDEX_REG` write | yes | yes |
| Count `CTRL_ID_REG` read | yes | yes |
| Print `control[N] id=...` | yes | yes |
| Print `Detected N controls` | yes | yes |
| Descriptor `CTRL_INDEX_REG` write | yes | yes |
| Descriptor `CTRL_VALUE_REG <- 0` write | no | yes |
| Descriptor `SYSTEM_IDLE_REG` wait | no | yes |
| Descriptor `CTRL_ID_REG` read | yes | yes |
| Descriptor `CTRL_MAX_REG` read | yes | yes |
| Descriptor `CTRL_MIN_REG` read | yes | yes |
| Descriptor `CTRL_DEF_REG` read | yes | yes |
| Descriptor `CTRL_STEP_REG` read | yes | yes |
| Final `CTRL_INDEX_REG <- 0` reset checked | yes | yes |

## Git-history investigation

Raspberry Pi history shows `CTRL_VALUE_REG = 0` during enumeration and `SYSTEM_IDLE_REG` polling were present in the initial Pivariety driver import, not added later as an isolated fix:

| Branch inspected | Earliest relevant commit | Date | Message | Evidence |
| --- | --- | --- | --- | --- |
| `rpi-5.15.y` | `723544dfd6d1` | 2022-04-14 | `media: i2c: Add driver of Arducam Pivariety series camera` | Initial file contains `CTRL_VALUE_REG`, `SYSTEM_IDLE_REG`, `wait_for_free()`, and enumeration-time `CTRL_VALUE_REG = 0` + `wait_for_free()` |
| `rpi-6.1.y` | `7078fe8ebf77` | 2022-04-14 | `media: i2c: Add driver of Arducam Pivariety series camera` | Same initial protocol operations present |
| `rpi-6.6.y` | `b7b65d3097fc` | 2022-04-14 | `media: i2c: Add driver of Arducam Pivariety series camera` | Same initial protocol operations present |

No commit message, PR text, or issue found in the inspected Raspberry Pi commit history explains these operations as a later bug fix. The available evidence classifies them as part of the original Pivariety firmware protocol imported with the driver, likely for Pivariety firmware synchronization/newer firmware behavior relative to the legacy 5.4 OBISP driver.

## Evidence-based conclusion template

Until target hardware logs are collected, the most likely failure class is **firmware protocol mismatch** with **Medium** confidence, because the confirmed failure occurs after driver match and during Pivariety enumeration, and the biggest runtime difference from legacy OBISP is the additional Pivariety `CTRL_VALUE_REG = 0` plus `SYSTEM_IDLE_REG` wait sequence.

Use the new logs to classify the first failing transaction exactly:

- Timeout waiting for `SYSTEM_IDLE_REG`: first failing / non-idle repeated lines are `step = read SYSTEM_IDLE_REG` after `CTRL_VALUE_REG <- 0`.
- Failed `CTRL_VALUE_REG` write: first `ENUM FAILED` reason is `descriptor value reset write failed`.
- Failed descriptor read: first `ENUM FAILED` reason is a `read CTRL_*_REG` descriptor read.
- Invalid control count: count discovery logs show zero, unreasonable, or early `NO_DATA_AVAILABLE` despite expected controls.
- Firmware protocol mismatch: legacy succeeds through descriptor reads while Pivariety fails on `CTRL_VALUE_REG` or `SYSTEM_IDLE_REG`.
