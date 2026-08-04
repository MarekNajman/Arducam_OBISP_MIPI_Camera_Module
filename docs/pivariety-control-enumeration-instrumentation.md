# Pivariety Control Enumeration Instrumentation Patch

`patches/instrument-pivariety-enum-controls.patch` is a diagnostic patch for the Raspberry Pi kernel source file:

```text
drivers/media/i2c/arducam-pivariety.c
```

It instruments only `pivariety_enum_controls()` so the first failing operation behind the probe-time message `enum controls failed.` can be identified.

## What it prints

The patch prints:

- detected firmware control count;
- the index currently being counted or parsed;
- each `CTRL_INDEX_REG` write and returned errno;
- each `CTRL_ID_REG`, `CTRL_MAX_REG`, `CTRL_MIN_REG`, `CTRL_DEF_REG`, and `CTRL_STEP_REG` read and returned errno;
- each `CTRL_VALUE_REG = 0` write and returned errno;
- each `SYSTEM_IDLE_REG` read performed by the wait loop used during control parsing;
- cumulative and first returned errno when the descriptor-read sequence fails;
- V4L2 control-handler, fwnode parse, and fwnode-property creation return codes;
- the final `-ENODEV` return path including the last parsed index.

## Behavioural intent

The patch is intended to be diagnostic only. It keeps the same firmware register sequence as the original driver:

1. Count controls by writing `CTRL_INDEX_REG` and reading `CTRL_ID_REG` until `NO_DATA_AVAILABLE`.
2. For each descriptor index, write `CTRL_INDEX_REG`, write `CTRL_VALUE_REG = 0`, wait for firmware idle, then read the descriptor registers.
3. Reset `CTRL_INDEX_REG` to 0.

The count-discovery loop is expanded inline so both the write and read return codes are visible; it performs the same logical operations as `pivariety_get_length_of_set(pivariety, CTRL_INDEX_REG, CTRL_ID_REG)`.

## Applying the patch

From a Raspberry Pi kernel tree checked out at the matching branch, run:

```sh
patch -p1 < /path/to/patches/instrument-pivariety-enum-controls.patch
```

Then rebuild and install the kernel module or kernel using the same process used for the in-tree `arducam-pivariety` driver.
