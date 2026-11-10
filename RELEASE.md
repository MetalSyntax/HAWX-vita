# Tom Clancy's H.A.W.X (PS Vita) — Preview Release v1.0

First public release of the native iOS **Mach-O loader** port of *Tom Clancy's H.A.W.X* (Gameloft, iPhone) for the PlayStation Vita. This is an early **work-in-progress** build: on real hardware the game boots, reaches the title screen and the menus, and creates a pilot profile. Missions have not been played on console yet.

---

## 📦 Release Package Contents

1. **`hawx.vpk`** (v01.00): PlayStation Vita application package with the Mach-O loader, Objective-C/UIKit runtime, vitaGL renderer, OpenAL/AudioQueue audio layer and touch/button input.
2. **`rebase.bin`**: the **77,629** absolute pointers of the executable that must be moved to the load address.
3. **`binds.bin`**: the **8,267** import slots of the executable and the Apple framework symbol each one calls.
4. **`nib.txt`**: `MainWindow.nib` precompiled to text (window and `EAGLView` setup).

None of these files contain game code or assets.

---

## 🛠️ What are rebase.bin and binds.bin?

`Hawx` is a non-PIE armv6 executable linked at a fixed address. The loader maps it at `0x98000000`:

- `rebase.bin` (`RBS1`, count, then one address per word) lists the 32-bit words in the executable that hold absolute addresses and must be moved to the new base.
- `binds.bin` (`BND1`) maps each import slot to its external symbol, which the loader resolves to its own native implementation.

Both are generated from your own executable with:

```bash
python3 -I tools/ios_prep.py ux0_data/hawx/game_app/Hawx ux0_data/hawx   # run by tools/stage_data.sh
```

They only match **version 1.1.3** of the game.

---

## 🚀 Player Installation Guide

1. **PS Vita requirements**:
   - Custom Firmware (**Enso 3.60 / 3.65** recommended).
   - `kubridge.skprx` under `*KERNEL` in `ur0:tai/config.txt`.
   - `libshacccg.suprx` in `ur0:data/` (obtainable via **ShaRKBR33D**).

2. **Game assets**:
   - Extract `Payload/Hawx.app/` from your decrypted iOS **1.1.3** `.ipa` (FairPlay `cryptid 0`).
   - Copy its contents to `ux0:data/hawx/game_app/`.
   - Copy `rebase.bin`, `binds.bin` and `nib.txt` from this release to `ux0:data/hawx/`.
   - Layout:
     ```text
     ux0:data/hawx/
     ├── game_app/      (contents of Hawx.app: Hawx, data.bar, *.swf, Info.plist, ...)
     ├── rebase.bin
     ├── binds.bin
     ├── nib.txt
     ├── controls.txt   (optional, see Controls)
     ├── Documents/     (created on first launch: profiles and settings)
     └── logs/          (created on first launch)
     ```

3. **Install**: install `hawx.vpk` with VitaShell and launch it from the LiveArea.

---

## 🎮 Controls

Physical buttons are translated into touches at fixed points, set in `ux0:data/hawx/controls.txt` (`BUTTON X Y`, 480×320 game coordinates, up to 6 lines). Without that file only touch and the left stick work.

| Input | Action | Status |
| :---: | :--- | :---: |
| **Front Touchscreen** | iPhone touch screen (menus, HUD) | ✅ Working |
| **Left Stick** | Tilt (simulated accelerometer: roll / pitch) | ⚠️ Not tested in flight |
| **Cross / Circle / Square / Start** | Touch at the point set in `controls.txt` | ⚠️ Not tested in flight |
| **Select + Start (hold 2 s)** | Cycle the picture rotation (saved to `rotation.txt`) | ✅ Working |

---

## 📋 Changelog (v1.0)

- **Loader**: 12-byte `__symbol_stub4` stubs were not rebased → crash in the first global constructor; now all 191 constructors run.
- **Weak C++ symbols**: external relocations against the game's own vtables/typeinfo added the value twice → corrupted vtables; the word is now used as-is.
- **Rebase heuristics**: sizes and GL enums passed to `new`/`malloc`/`gl*`, `zlib`'s `lbase` table and `GL_TEXTURE0` were mistaken for pointers; Gameloft string literals (`&str+13`) were missed. Final count: 77,629 words.
- **NSString lifetime**: `UTF8String` buffers freed by an autorelease pool were reused by `fopen` → garbage `Documents/` paths; buffers are now quarantined before being freed.
- **Orientation**: `UIDevice orientation` now matches the displayed rotation → no upside-down picture or 3↔4 flicker.
- **Touch**: real hit-testing per view (touches no longer go to an invisible `UITextField`).
- **Keyboard**: `sceImeDialog` text is delivered through `textField:shouldChangeCharactersInRange:replacementString:`, and Return is sent 0.5 s later so the game validates the name → profile creation works.
- **Touch release**: an `ended` touch keeps its last position instead of (-1,-1), so Flash buttons register the click.

---

## ⚠️ Known Issues

- **Early build**: only boot, title screen, menus and profile creation were tested on hardware. Missions, audio and in-flight controls are **not verified on console**.
- **Menu "next" button**: the touch-release fix above is included but not verified on console yet.
- **OpenAL log warning**: `Error creating device/context` appears in the log even though a device is opened.
- **Side bars**: the picture is shown at 816×544 (aspect kept), not stretched.
- **Online features**: Gameloft online services and Game Center are not available.

Installation and build instructions: see [README.md](README.md).
