# DOSBox ReelMagic — Copilot Workspace Instructions

## Project Overview

This is **dosboxrm** — a fork of DOSBox 0.74-3 that emulates the Sigma Designs ReelMagic MPEG decoder card for DOS gaming. The primary user (Roland) runs Return to Zork ReelMagic Edition on an **Apple M4 Mac Mini** with a **5K LG UltraFine display** (5120x2880) and a **genuine Roland MT-32** synth connected via CoreMIDI (UM-ONE USB MIDI adapter, destination index 0).

## Codebase Conventions

- **Language:** C/C++ (C89-compatible C++ with some C++ features: classes, exceptions, namespaces)
- **Build system:** GNU Autotools (autoconf/automake). Always `make` from `dosbox-0.74-3/`.
- **No STL containers** in hot paths — raw arrays and C-style memory management
- **Logging:** Use `LOG(LOG_REELMAGIC, LOG_NORMAL|LOG_WARN|LOG_ERROR)(fmt, ...)` for ReelMagic code
- **Config types:** `Bit8u`, `Bit16u`, `Bit32u`, `Bitu` (not `uint8_t` etc.) — DOSBox convention
- **PL_MPEG is header-only** — `reelmagic_pl_mpeg.h` is the implementation (included via `reelmagic_pl_mpeg.cpp` with `#define PL_MPEG_IMPLEMENTATION`)

## Key Architecture

```
reelmagic_driver.cpp   → RMDEV.SYS + FMPDRV.EXE emulation (INT 2F + INT 80h+ API)
reelmagic_player.cpp   → MPEG decode, delta/delta f_code, audio FIFO, streaming
reelmagic_videomixer.cpp → VGA + MPEG compositing with z-order, alpha, DUP5
reelmagic_pl_mpeg.h    → Modified PL_MPEG with decode_picture_header_callback hook
sdlmain.cpp            → SDL display, OpenGL rendering, integer-scale viewport
```

## Critical Details

### Magical MPEG Format
- ReelMagic MPEG files have corrupted f_code values in P/B picture headers
- Recovery uses **delta/delta algorithm** (pre-computed 56-entry table from magic key)
- Magic key `0x40044041` = most games (RTZ, LOTR), `0xC39D7088` = The Horde
- Frame rate recovery: `magical_rate_code & 0x07`
- Full documentation in `NOTES_MPEG.md`

### Video Mixer
- VGA output is intercepted before DOSBox RENDER via `vga_reelmagic_override.h`
- MPEG surface z-order: 1=invisible, 2=in front of VGA, 4=behind VGA
- Alpha transparency: palette index 0 or pure black in BGRA pixel format
- DUP5 hack: duplicates every 5th VGA line (320x200 → 320x240) for 4:3 aspect
- PlayerPicturePixel is BGRA (4 bytes), matching RenderOutputPixel

### Display Pipeline
- OpenGL fixed-function (no shaders) — SDL1.2 via sdl12-compat on macOS
- Integer-scale viewport in fullscreen: `min(surfW/width, surfH/height)` centered
- Power-of-2 texture sizes for compatibility
- Nearest-neighbor filtering (`openglnb`) for pixel-perfect output

### Build on macOS ARM64
```bash
brew install autoconf automake libtool sdl12-compat
cd dosbox-0.74-3 && bash autogen.sh && ./configure && make -j$(sysctl -n hw.ncpu)
```
- `configure.ac` has `aarch64|arm64|arm*` case → `C_TARGETCPU=ARMV8LE`, unaligned memory
- Binary: `src/dosbox` (Mach-O 64-bit executable arm64)

### Testing
```bash
# Run with specific config:
./src/dosbox -conf /path/to/DOSBoxRTZ-RM.conf

# Debug build:
./configure --enable-debug=heavy && make -j$(sysctl -n hw.ncpu)
```

### Game Config Location
Roland's RTZ config: `/Users/roland/Games/DOSBox/REELMAGIC/Reelmagic/DOSBoxRTZ-RM.conf`
Game files: `/Users/roland/Games/DOSBox/REELMAGIC/Reelmagic/`

## Common Pitfalls

- `plm_rewind()` requires seekable buffer — don't use in streaming code paths
- VGA DUP5 functions must handle both MPEG-present and VGA-only cases
- `_magicalFcodeOverride` (magicfhack config) is debug-only — uses static callback, not delta/delta
- SDL12-compat on macOS doesn't redirect stdout/stderr to files like classic SDL1
- After editing `configure.ac`, must run `bash autogen.sh` before `./configure`
- `make clean` doesn't remove `.a` archive files — remove manually if needed

## Related Documentation

| File | Contents |
|------|----------|
| `NOTES_MPEG.md` | Magical MPEG format, f_code corruption, delta/delta algorithm |
| `RMDOS_API.md` | Full reverse-engineered ReelMagic DOS driver API |
| `RMPORTIO_NOTES.md` | Physical board port I/O and PL-450 firmware loading |
| `KNOWN_GAME_BUGS.md` | Hardware-accurate bugs (not emulator defects) |
| `IMPROVEMENTS.md` | Display/video/audio quality analysis and recommendations |
