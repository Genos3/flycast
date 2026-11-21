# Debugging helpers

This fork adds two small file-based helpers that make it possible to debug a
Dreamcast guest without reading the GUI. Both write into the **current working
directory** of the emulator process, so launch Flycast from a scratch folder
(or from `build/`) and look there for the output.

## 1. Guest serial output -> `serial.log`

Every byte the guest writes to the SCIF serial port is also appended to
`serial.log` in the working directory. The file is opened in append mode and
flushed on every newline, so it can be read while the emulator is still
running.

Enable it in `emu.cfg`:

```ini
[config]
Debug.SerialConsoleEnabled = 1
```

On Windows this build is portable, so `emu.cfg` lives next to `flycast.exe`
(i.e. in `build/`) and is created automatically on the first run.

Notes:
- Lines usually appear twice: once with a `[DC]` prefix (the KOS debug-log
  layer) and once raw. That is expected.
- The usual GUI "Flycast Serial Output" console window still appears; the log
  file is an addition, not a replacement.
- `SerialPTY` works as an alternative switch on other platforms.

## 2. Guest frame screenshots -> `shot_00030.png`, ...

Set the environment variable `FLYCAST_SHOT_FRAMES` to a comma-separated list
of **presented-frame** numbers before launching Flycast. Each requested frame
is written as `shot_<frame>.png` in the working directory.

Windows (PowerShell):

```powershell
$env:FLYCAST_SHOT_FRAMES = "1,2,3,5,10"
.\flycast.exe "C:\path\to\game.cdi"
```

Bash / MSYS2:

```bash
FLYCAST_SHOT_FRAMES=1,2,3,5 ./flycast.exe game.cdi
```

Notes:
- Frames count calls to `Present()`, not the guest PVR frame counter. A 2D-only
  guest may present only a handful of frames in the first tens of seconds
  (the Mu Online DC port presents about five times in 20 s), so use small
  numbers. If a requested frame is skipped, the next presented frame is used.
- The capture is the renderer's last guest framebuffer, at full resolution —
  no emulator UI or overlay is included.
- For guests whose 2D path leaves the offscreen framebuffer empty, this
  captures black frames. In that case use an OS-level window capture instead
  (e.g. Win32 `GetWindowRect` + `Graphics.CopyFromScreen` on the game window).
- Works with every renderer implementing `GetLastFrame()` (GLES, Vulkan, DX9,
  DX11). Screenshots use the default renderer output orientation.

## Typical debugging run

```powershell
cd C:\...\flycast\build
$env:FLYCAST_SHOT_FRAMES = "1,2,3,5,10"
.\flycast.exe "C:\...\mu_full.cdi"
```

Then, while it runs:

```powershell
Get-Content .\serial.log -Tail 40
Get-ChildItem .\shot_*.png
```

## Build notes

Configure/build with MSYS2 mingw64 + Ninja (2 parallel jobs):

```bash
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DCMAKE_C_COMPILER=/mingw64/bin/gcc.exe \
  -DCMAKE_CXX_COMPILER=/mingw64/bin/g++.exe -DUSE_DISCORD=OFF
cmake --build build --parallel 2
```
