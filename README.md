# Monsgeek M1V5 VIAL Port
**By Poncho — V1.0**

> *This firmware is dedicated to my best friends, Ben (ElitePie117) and Thomas. Their names and numbers are encoded in the keyboard's UID: `0x13, 0x37, 0x31, 0x44, 0x13, 0x64, 0x54, 0x42` — 1337, 314, 413, 64, T, B.*

---

## What Is This?

This is a community port of the Monsgeek M1V5 keyboard from VIA to VIAL, with full tri-mode wireless connectivity (USB, Bluetooth, 2.4GHz) preserved. As far as I can tell, this is the first successful public port. Previous attempts that I've read about have failed because they used source files from the wired-only M1 V1, which bricks the wireless co-processor.

VIAL gives you real-time key remapping and RGB control like VIA, but it also includes tap dance, combos, and key overrides without needing to reflash after every little change.

This is for the US version of the keyboard. If I'm right, the patches applied should work for the UK version of the board as well, the only difference is that the visuals in VIAL will be wrong.

---

## Quick Start — Just Want to Flash It?

### What You'll Need
- [QMK Toolbox](https://github.com/qmk/qmk_toolbox/releases) (latest stable)
- [Vial app](https://get.vial.today) (Windows/Mac/Linux)
- The `.bin` file from the this repo
- Your M1V5's recovery firmware from [MonsGeek's support page](https://www.monsgeek.com/faq/how-to-fix-double-tapping-issue-on-m1-v5-via-keyboard/), keep it as a backup. (Why they have it in such a weird page is beyond me)

### Before You Flash
- Make sure the switch under Caps Lock is in the **middle** position
- Run QMK Toolbox as Administrator on first launch to download drivers
- If it doesn't automatically install drivers on admin launch, in the application, go to tools -> Install Drivers...

### Flashing
1. Open QMK Toolbox and load the `.bin` file
2. Unplug your keyboard
3. Hold **ESC** and plug the USB cable back in
4. QMK Toolbox should show: `WB32 DFU device connected`
5. Click **Flash**
6. Wait for `Flash complete!`
7. Unplug and replug normally

### Using Vial
When you open the Vial app, your keyboard should be detected automatically as "AKKO M1 V5." From there you can do what you like with it. There's a ton more functions in VIAL than there is in VIA, and if you ever have questions, QMK documentation should be able to point you in the right direction. 

---

## Known Limitations (V1)

The `rgb_record` feature, a Westberry-writen RGB system that MonsGeek used to build functions on top of, is disabled in V1 due to EEPROM API incompatibilities between MonsGeek's QMK fork and vial-qmk. I thought it would be easier to disable it than to port it, and when I disabled it, I didn't have as much of a grasp of the architecture as I do now. By the time I realized my mistake, it was more practical to finish V1 than it was to address it properly, which I'll do in V2. Disabling this affects:

- RGB mode cycling via FN+DEL. It now cycles stock QMK effects instead of Monsgeek's custom set
- Battery level indicator via FN+Space. It technically queries the battery now, but doesn't display anything on account of `rgb_record` being disabled
- The RGB record keycodes (RP_P0, RP_P1, RP_P2, RP_END) are present but do nothing

**V2 will restore rgb_record functionality.** If you fix it before I do, please open a PR. The relevant files are `rgb_record/rgb_record.c` and the `config.h` inside of the `m1_v5_us` folder. The core issue is that `EECONFIG_USER_DATABLOCK` was removed from vial-qmk in favor of `eeconfig_read_user_datablock()` / `eeconfig_update_user_datablock()`.

---

## Recovery

If something goes wrong, flash the official MonsGeek VIA firmware from their support page. As long as you can enter DFU mode (hold ESC + plug in), the keyboard is recoverable.

As a last resort, you can short the two pads on the spacebar PCB with tweezers while plugging in to force DFU mode. I've personally never done this before, so I wouldn't know exactly how it's done. Good luck.

**Never flash M1 V1 firmware onto the M1 V5.** The V1 firmware doesn't know the wireless co-processor exists and can corrupt it permanently.

---

---

# Technical Documentation

*Everything below is for those who wish to know the nitty-gritty details of this port for the sake of development or curiosity. If you just wanted to flash the firmware, you should be good.*

---

## Hardware Architecture

The M1V5 uses a **dual-chip design**:

- **Main MCU**: Westberry WB32FQ95, it runs QMK, handles the key matrix, RGB, and USB HID
- **Wireless co-processor**: A separate chip communicating with the main MCU over UART (TX: C10, RX: C11), it handles Bluetooth and 2.4GHz

When you flash QMK firmware you are only touching the main MCU. The co-processor has its own firmware that is not publicly available. This is why flashing the wrong firmware can permanently break wireless. Incorrect GPIO configuration on the main MCU can corrupt the co-processor's communication lines.

---

## Why This Port Was Difficult

MonsGeek maintains their own fork of QMK at `github.com/MonsGeek/qmk_firmware`, but all of the files for the wireless versions of the keyboards are on the `wireless` branch. This fork was written against an older version of QMK. VIAL is built on a newer version of QMK with different APIs. Porting requires transplanting MonsGeek's keyboard-specific code into a different host codebase, which means resolving every API incompatibility by hand.

Previous port attempts failed because they used `keyboards/monsgeek/m1` from the `master` branch, the wired-only V1 board. The V5 keyboard is on the `wireless` branch at `keyboards/monsgeek/m1_v5/m1_v5_us`.

---

## Environment Setup

### Prerequisites
- Windows with WSL2, or some linux distro. I used Ubuntu (I used WSL for the project, which is why I recommend using it)
- Git, Python 3, pip

### Step 1: Install QMK toolchain in WSL

```
pip install qmk --break-system-packages
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
qmk setup 
sudo apt install -y gcc-arm-none-eabi binutils-arm-none-eabi dfu-util gcc-avr avrdude dfu-programmer dos2unix avr-libc
```

>Note: `qmk doctor` may report warnings related to AVR if you're using Ubuntu like I did, that's because Ubuntu doesn't have the standard library headers, you'll need to get those separately. The M1V5 uses ARM architecture *anyway*, so unless you plan on wokring on AVR related projects, you should be fine.


### Step 2: Clone vial-qmk
```
git clone https://github.com/vial-kb/vial-qmk.git ~/vial-qmk --depth 1
cd ~/vial-qmk && make git-submodule
```

### Step 3: Clone MonsGeek's wireless branch
```
git clone https://github.com/MonsGeek/qmk_firmware.git --branch wireless --single-branch ~/monsgeek_wireless
```

### Step 4: Copy keyboard files into vial-qmk

```
cp -r ~/monsgeek_wireless/keyboards/monsgeek/m1_v5 ~/vial-qmk/keyboards/monsgeek/
cp -r ~/monsgeek_wireless/keyboards/linker ~/vial-qmk/keyboards/
```
The `linker/wireless/` folder is super important since it's the actual wireless driver used by the M1V5 (referenced in `post_rules.mk`). The `monsgeek/wireless/` folder is used by other boards (m1w, m2_v5) and should not be confused with it. 

### Step 5: Clone this repo and apply patches
```
git clone https://github.com/StevenVillarreal413/M1V5-VIAL-Port.git
cp -r M1V5-VIAL-Port/keyboards ~/vial-qmk/
```
This step copies patched files and vial keymap into the correct locations. It will only overwrite files that needed changes for the sake of the port. If you're curious about what changed and why, please check the API Incompatibilities section below.
### Step 6: Build
```
cd ~/vial-qmk
qmk compile -kb monsgeek/m1_v5/m1_v5_us -km vial
```

The output binary will be at `~/vial-qmk/monsgeek_m1_v5_m1_v5_us_vial.bin`.

---

## Key Files and What They Do

### `post_rules.mk`
Pulls in two external drivers at build time:
```
include keyboards/monsgeek/m1_v5/m1_v5_us/wls/wls.mk
include keyboards/linker/wireless/wireless.mk
```
`wireless.mk` defines `WIRELESS_ENABLE`, which activates all `#ifdef WIRELESS_ENABLE` blocks in `m1_v5_us.c`. Without this file the wireless keycodes compile out entirely.

### `wls/wls.c` and `wls/wls.h`
The hardware abstraction layer between QMK and the wireless co-processor. Handles physical switch detection (which mode the hardware switch is in), RGB status indicators for connection state, sleep/wake power management, and UART interrupt callbacks.

### `linker/wireless/`
The core wireless driver. Manages UART communication with the co-processor, transport switching between USB and wireless HID, low power modes, and the message protocol (`smsg.c`, `transport.c`, `md_raw.c`). This is a more evolved version of `monsgeek/wireless/`. This was confirmed when I checked using `diff`, which showed added pairing state checks, simplified NKRO handling, and a USB polling rate test function.

### `m1_v5_us.c`
The main keyboard behavior file. Defines all custom keycode handlers including the wireless keycodes (`KC_BT1/2/3`, `KC_2G4`, `KC_USB`), RGB controls, battery query, Mac/Windows layer switching, and the WASD↔Arrow direction swap (`HS_DIR`).

### `linker/wireless/md_raw.h`
Intercepts `raw_hid_send` calls to route HID communication through the wireless driver. Uses a line-number macro trick to redirect calls at specific lines in `via.c`. **If you update vial-qmk, check this file** — the line numbers may need updating if `via.c` changes.

---

## API Incompatibilities Fixed

These are the differences between MonsGeek's QMK fork and vial-qmk that required manual resolution:

| Issue | MonsGeek fork | Fix |
|-------|--------------|-----|
| `keyboard_protocol` variable | Existed in fork | Line deleted from `transport.c`, not present in vial-qmk |
| `EECONFIG_USER_DATABLOCK` | Raw EEPROM address pointer | Disabled in V1 with rgb_record, please see Known Limitations |
| `eeconfig_read_keymap()` | Returns value directly | Takes pointer: `eeconfig_read_keymap(&keymap_config)` |
| `eeconfig_update_keymap(raw)` | Takes raw uint16 value | Takes pointer: `eeconfig_update_keymap(&keymap_config)` |
| `UART_TX_PIN` / `UART_RX_PIN` | Was not explicitly defined, fell through to uart_serial.c defaults (A9/A10) | Added to `config.h` as C10/C11, the actual physical board pins |
| `UART_RX_PAL_MODE` | Were not explicitly defined, fell through to default | Added to `config.h` as 7, the alternate function mode for UART on WB32 |
| `UART_DRIVER` | Defaulted to SD1 in uart_serial.c | Added to `config.h` as SD3, which is the UART peripheral the M1V5 actually uses |
| `matrix_previous[]` | Global array accessible via extern | Replaced with local array in `lowpower.c` |
| `raw_hid_send` line numbers | Hardcoded to via.c line 461 in `md_raw.h` | Added lines 442 and 466 for vial-qmk's via.c|
| `rgb_record` EEPROM API | `EECONFIG_USER_DATABLOCK` pointer | Disabled in V1 — see Known Limitations |

---

## The Vial UID

The keyboard's Vial UID is `{0x13, 0x37, 0x31, 0x44, 0x13, 0x64, 0x54, 0x42}`.

This decodes as: `1337` (The "Elite" in ElitePie117) + `314` (Thomas's favorite number and the Pie in "ElitePie117") `413` (Homestuck reference for our favorite webcomic) + `64` (My favorite number) + `T` (Thomas) + `B` (Ben).

---

## Changelog

### V1.0
- Initial release
- Full tri-mode wireless (USB, Bluetooth, 2.4GHz)
- Vial Utility
- rgb_record disabled pending API port

### V2.0 (planned)
- Restore rgb_record functionality
- Battery indicator
- Monsgeek custom RGB effects via FN+DEL

---

## Credits

- **Poncho** - port, debugging, documentation
- **MonsGeek / yangzheng20003** - original VIA firmware and wireless driver
- **Westberry Technology** - WB32FQ95 MCU and rgb_record library
- **vial-kb** - VIAL firmware and app

---

## Support

If this saved you time or a bricked keyboard, consider buying me a coffee:

https://ko-fi.com/stevenvillarreal64

Issues and PRs welcome, especially for V2 rgb_record work.
