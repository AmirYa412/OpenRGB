# Galax GPU v2 Controller — Bug Report & Fix Attempts

## Bug
OpenRGB bricks 2 of 3 fan LEDs on Galax 5070 Ti (v2 controller, I2C address `0x51`).
Only 1 fan responds after first use. Xtreme Tuner also cannot recover the fans afterward.

## Root Cause of Bricking
The original code wrote RGB as three separate 1-byte writes to registers `0x02`, `0x03`, `0x04`.
The hardware requires a **single 3-byte I2C block write** to register `0x02`.
The single-byte writes corrupted the per-fan state, and a subsequent save (`0x40=0x5A`)
persisted the corruption to EEPROM.

---

## Fixes Applied
*(All in commits `fde29c6c` through `34b7a450`)*

| # | Fix | Description |
|---|-----|-------------|
| 1 | **3-byte block write** | Changed `SetLEDColors()` to use `i2c_smbus_write_i2c_block_data` for registers `0x02–0x04` in one transaction |
| 2 | **Brightness clamping** | Controller reads back `0xFF`/`0x81` for uninitialized registers; now clamped to valid range (`0x00–0x03`) |
| 3 | **Speed clamping** | Clamped to `0x00–0x09` |
| 4 | **Fan select all (`0x2B=0x00`)** | Added `SetFanSelectAll()` write after mode and before brightness, matching Xtreme Tuner sequence |
| 5 | **Speed registers skipped for static mode** | `0x21`/`0x22`/`0x23` only written for Breathing/Rainbow modes |
| 6 | **Color sent before save** | `DeviceSaveMode()` now calls `DeviceUpdateLEDs()` before `DeviceUpdateMode()` |

---

## Recovery Attempts (Bricked Unit)

### EEPROM Recovery
Wrote full Xtreme Tuner static white sequence (`color → mode → fan select → brightness → save`) on startup.
**Result:** 1 fan turns white, 2 stay dead.

### Per-Fan Channel Recovery
Wrote save sequence with `0x2B=0x01`, `0x02`, `0x03` individually to try addressing each fan.
**Result:** Awaiting test — likely won't work.

---

## Key Findings

- All I2C writes return success (`ret=0`)
- All registers read back as `0xFF` — controller may not support reads at all
- Debug logging confirmed `DeviceSaveMode()` override is called correctly, `colors.size()=1`, correct RGB values sent
- The fixes produce the **exact same I2C sequence** as Xtreme Tuner for static mode
- The bricked fans **cannot be recovered by Xtreme Tuner either** — likely needs firmware flash from Galax

---

## Action Items

### For the Tester's Card
> Contact Galax support for a firmware flash/reset of the LED controller.

### Before Merging Fixes for New Cards
1. **Remove** the recovery/debug code (recovery on init, `LOG_DEBUG` lines)
2. The core fixes (block write, clamping, fan select, speed gating, save sequence) are **correct and safe**
3. ⚠️ **Needs testing on a working Galax 5070 Ti before shipping** — all testing was done on the already-bricked unit
