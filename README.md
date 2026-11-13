# Autorun F.E.A.R. 2 Fixed

Fixes that get **F.E.A.R. 2: Project Origin** (PC, 32-bit) running on Nintendo Switch with [Autorun](https://github.com/autorunhq/autorun) 258, the Wine-based Windows runtime for Horizon.

This repository does **not** contain any game files. It only holds Wine DLLs rebuilt from Autorun's source code, configuration files, source patches and documentation of how the fixes were found.

**Download:** grab the ready-to-copy package from the [Releases](https://github.com/ipungsx/Autorun_FEAR_2_Fixed/releases) page.

## Problems and fixes

| # | Symptom | Root cause | Fix | Status |
|---|---------|------------|-----|--------|
| 1 | The game crashes in `xaudio2_3.dll` when audio starts | Autorun 258's audio driver rejects the game's 5.1 (6-channel) stream, and FAudio then calls `Release()` on a NULL pointer | New `xaudio2_3.dll`: when a multichannel format is refused, the device is opened in stereo and the 6-channel mix is folded down to stereo | Tested |
| 2 | Black screen or crash with any DXVK other than Sarek | `IDirect3DDevice9::Reset` fails in DXVK 1.10.3 and official DXVK ("Remaining resources: 7") | Select **DXVK Sarek** in Autorun's settings | Tested |
| 3 | "Sound Acceleration Error" dialog, then a long black screen | The IMA ADPCM ACM codec is not registered, but the engine requires IMA ADPCM, MP3 and PCM codecs | A `C:\windows\system.ini` that registers Wine's built-in ACM codecs | Tested, the game reaches gameplay |
| 4 | Left and right sticks do nothing | The PC version of F.E.A.R. 2 does not read gamepads at all, and Autorun 258 uses an older control-file format than current Autorun | `FEAR2.keys.txt` (old format): left stick = W A S D, right stick = mouse | Tested for the left stick and buttons |
| 5 | The right stick does not turn the camera; the camera sometimes jumps or spins on its own | Wine 11.18 collects relative mouse motion, and Autorun 258's display driver never sends it on until a click happens | New `user32.dll`: flushes the collected motion (all 32-bit programs) and handles the cursor recentring of `FEAR2.exe` | **Not yet confirmed on hardware** |

The full write-up of every step, the log evidence and the reasoning behind each decision is in [docs/PROSES.md](docs/PROSES.md) (Indonesian).

## Repository layout

```
sdcard/                         copy the contents of this folder to the SD card root
  switch/wine/drive_c/
    windows/system.ini          ACM codec registration
    windows/syswow64/xaudio2_3.dll
    windows/syswow64/user32.dll
    FEAR 2 Project Origin Complete/FEAR2.keys.txt
patches/                        source changes against Autorun
  0001-faudio-xaudio2-refused-stream-stereo-fold.patch
  0002-user32-mouse-motion-flush-and-fear2-mouselook.patch
  diagnostics/                  logging-only patches used during the analysis
docs/PROSES.md                  analysis and fix process (Indonesian)
docs/BUILD.md                   how to rebuild the DLLs from source
licenses/                       Wine (LGPL 2.1) and FAudio (zlib) licenses
CREDITS.md                      credits
CHECKSUMS.sha256                checksums of every file in sdcard/
```

## Installation

1. Close Autorun.
2. Copy the `switch` folder (from the release zip, or from `sdcard/` in this repository) to the **root** of the SD card, overwriting existing files.
   - If `sdmc:/switch/wine/drive_c/windows/system.ini` already exists, do not overwrite it. Instead, add the lines under `[drivers32]` from this repository's file to it.
   - `FEAR2.keys.txt` must sit **next to `FEAR2.exe`**. If your game folder is not named `FEAR 2 Project Origin Complete`, move the file into the right folder. If a `FEAR2.keys.txt` is already there, rename it first as a backup.
3. In Autorun, open the settings for F.E.A.R. 2:
   - **DXVK**: choose **Sarek**.
   - **Controls** (Diagnostics section): choose **Its own**, so that `FEAR2.keys.txt` is read.
4. Do not rename the `Game\interface\videos` folder. Leave the Bink videos as they are.
5. Start the game.

Do not run **Settings > System > Windows DLLs** or **Verify files** after installing. Both download the official DLLs again and overwrite this repository's `xaudio2_3.dll` and `user32.dll`. To **go back to the official state**, run **Verify files**, then delete `system.ini` if you wish.

### Control mapping (`FEAR2.keys.txt`)

| Switch control | Sent to the game |
|----------------|------------------|
| Left stick | W A S D |
| Right stick | Mouse movement (camera) |
| A / B | Left click (fire) / right click |
| ZR / ZL | Space (jump) / C (crouch) |
| X / Y | E (use) / R (reload) |
| L / R | G (grenade) / Shift |
| Left / right stick click | Ctrl / F |
| D-pad | 1, 2, 3, 4 (weapons) |
| + / - | Esc / Tab |

What each key does in game can be changed in F.E.A.R. 2's own **Options > Controls**. The file format is `NAME=code`, where the code is a Windows virtual-key code in hexadecimal. `LSTICK` and `RSTICK` accept `keys` or `mouse`.

## Effect on other games

- `xaudio2_3.dll`: applies to every 32-bit game that uses XAudio2 2.3. Behaviour only changes when the audio driver refuses a multichannel format; in that case sound falls back to stereo instead of crashing or hanging.
- `system.ini`: registers codecs whose files Autorun already ships. Other games that need those codecs benefit as well.
- `user32.dll`:
  - Flushing collected mouse motion applies to **every 32-bit program**. When nothing has been collected, nothing is sent.
  - The cursor-recentring handling and the input logging are **only active for `FEAR2.exe`**.
- 64-bit programs are not affected, because they use `system32\user32.dll` (ARM64X).

## Limitations

- The camera fix (#5) has not been confirmed on hardware yet. Test reports are very welcome through *Issues*.
- Newer Autorun builds already include the official fix for flushing mouse motion (commit [`0cc936d1`](https://github.com/autorunhq/autorun/commit/0cc936d1863209af659977cdb001c8c6fdde94cc), 12 October 2026). If your Autorun includes that commit, this repository's `user32.dll` is not needed for that part.
- The DLLs are built from Autorun commit `1ac9b6cf` (16 October 2026) and tested on Autorun 258 with runtime build `nx-amd64-fex-2640`. Other combinations are untested.
- F.E.A.R. Perseus Mandate and F.E.A.R. Extraction Point are out of scope.

## Technical summary

1. **Audio crash.** Autorun 258's audio driver rejects F.E.A.R. 2's 6-channel float stream (`AUDCLNT_E_UNSUPPORTED_FORMAT`, `0x88890008`), after which FAudio released a NULL `IAudioRenderClient`. The patched `xaudio2_3.dll` retries in stereo and folds the 6-channel mix down (centre and surround channels at 0.707, LFE dropped). If no render client can be obtained at all, it keeps mixing in real time into a discarded buffer, so the voice callbacks the game waits for still fire.
2. **Black screen.** D3D9 `Reset` fails in the other bundled DXVK builds. A game-folder `dxvk.conf` is not read by Autorun 258, so `d3d9.countLosableResources` cannot be used; DXVK Sarek does not have the problem.
3. **"Sound Acceleration Error".** `GameClient.dll` shows this dialog when the engine's sound start-up returns `0x23`. In `FEAR2.exe`, the engine enumerates ACM drivers with `acmDriverEnum` and requires format tags `0x11` (IMA ADPCM), `0x55` (MPEG Layer-3) and `0x01` (PCM). Autorun's setup only registers `msacm.l3acm` and ships no `system.ini`, so `system.ini` registers Wine's built-in codecs under `[drivers32]`.
4. **Controls.** The PC game reads only the keyboard (window messages and `GetKeyboardState`) and the mouse (`WM_MOUSEMOVE` plus a `SetCursorPos` to the window centre every frame). This Autorun build reads `FEAR2.keys.txt` in the `NAME=code` format, and only when *Controls* is set to *Its own*. Its default left-stick mapping sends arrow keys.
5. **Camera.** Wine 11.18's `accum_mouse_motion` coalesces relative mouse motion per thread until a button, a wheel event or `MOUSEEVENTF_MOVE_NOCOALESCE` arrives, and this Autorun build's display driver never sends one. The patched `user32.dll` therefore:
   - after each `PeekMessageW`/`GetMessageW`, sends `MOUSEEVENTF_MOVE_NOCOALESCE` through `NtUserCallHwndParam_SendHardwareInput`, the same call the driver uses (with a 64-bit `INPUT` layout, because the WoW64 thunk passes the pointer through unchanged), at most once every 4 ms per thread;
   - for `FEAR2.exe` only, keeps the per-frame warps from moving the real cursor, rewrites mouse messages to the position the game expects, posts the `WM_MOUSEMOVE` that a real warp would have produced, and recentres the real cursor only near the screen edges.

## License

The DLLs in `sdcard/` are derived from Wine and FAudio. Wine is licensed under the GNU LGPL 2.1 (see `licenses/Wine-LGPL-2.1.txt`), and FAudio under the zlib license (see `licenses/FAudio-zlib.txt`). The complete source is [autorunhq/autorun](https://github.com/autorunhq/autorun) at commit `1ac9b6cf37b90f2d5783dd7ef389bebbe1c68877` plus the patches in `patches/`. Build instructions are in [docs/BUILD.md](docs/BUILD.md).

F.E.A.R. and F.E.A.R. 2: Project Origin are trademarks of their respective owners. This repository is not affiliated with the game's developer or publisher and does not distribute any game files.

Full credits are in [CREDITS.md](CREDITS.md).
