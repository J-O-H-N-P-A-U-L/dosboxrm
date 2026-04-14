
# DOSBox ReelMagic (dosboxrm)

A fork of DOSBox 0.74-3 that emulates the Sigma Designs ReelMagic MPEG decoder card,
enabling playback of ReelMagic-enhanced DOS games on modern hardware — including
native Apple Silicon (ARM64) Macs.

## Contributors

* Jon Dennis <jrdennisoss_at_gmail.com>
* Chris Guthrie <csguthrieoss_at_gmail.com>
* Joseph Whittaker <j.whittaker.us_at_ieee.org>

Built on DOSBox by the DOSBox Team and Dominic Szablewski's
[PL_MPEG](https://github.com/phoboslab/pl_mpeg) decoder library.


## Game Compatibility

The following ReelMagic games are known to work:

| Game | Status |
|------|--------|
| Return to Zork | Working (no subtitles — hardware-accurate) |
| Lord of the Rings | Working |
| The Horde | Working |
| Entity | Working (crashes on video message — hardware-accurate) |
| Man Enough | Working (no subtitles in first scene — hardware-accurate) |
| Dragon's Lair | Working |
| Flash Traffic | Working |
| Crime Patrol | Working |
| Crime Patrol 2 - Drug Wars | Working |

See `KNOWN_GAME_BUGS.md` for bugs that exist on real hardware too.


## Current State

### What Works
* Full ReelMagic driver emulation (`RMDEV.SYS` + `FMPDRV.EXE`) via `reelmagic_driver.cpp`
* MPEG-1 video decode with "magical" f_code recovery (delta/delta algorithm — see below)
* VGA + MPEG video compositing with z-ordering and alpha transparency
* MPEG audio decode and mixing via DOSBox mixer
* File mode and streaming mode MPEG playback
* Native ARM64 (Apple Silicon) builds
* OpenGL integer-scale viewport for pixel-perfect fullscreen on HiDPI displays
* DUP5 vertical line duplication with Bresenham midpoint distribution for 4:3 aspect

### Known Limitations
* MPEG DMA streaming is not fully implemented
* Port I/O to the physical ReelMagic board is not emulated (software driver emulation only)
* A handful of unknown driver API subfunctions remain unimplemented (see `RMDOS_API.md`)


---


# "Magical" MPEG-1 Format and Delta/Delta Decode

ReelMagic games encode their MPEG-1 video files with corrupted `f_code` values in
P and B picture headers — a form of "clone protection" that prevents standard MPEG
players (VLC, etc.) from rendering motion correctly. Only I-pictures and still
scenes decode correctly on a standard player; anything with motion vectors is garbled.

The emulator recovers the correct `f_code` per-picture using the **delta/delta
algorithm**: a pre-computed 56-entry lookup table derived from the game's "magic
key" (a 32-bit value provisioned via `driver_call(9, handle, 0x210, ...)`) that
provides a delta correction for each temporal sequence number (TSN). This runs at
decode time with zero file I/O — no seeking or scrubbing required.

Known magic keys:

| Key | Even Pattern | Games |
|-----|-------------|-------|
| `0x40044041` | [4, 3, 2, 3] | Return to Zork, Lord of the Rings, most games |
| `0xC39D7088` | [1, 3, 3, 3] | The Horde |

The frame rate code is recovered by: `MAGICAL_FRAME_RATE_CODE & 0x07`

Full details in `NOTES_MPEG.md`.


---


# Configuration

A `[reelmagic]` section in the DOSBox config file controls the emulator:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `enabled` | `true` | Enable/disable ReelMagic emulation |
| `alwaysresident` | `false` | Force `FMPDRV.EXE` to always be loaded (some games need this) |
| `vgadup5hack` | `false` | Duplicate every 5th VGA line for 4:3 aspect ratio (320x200 → 320x240) |
| `initialmagickey` | `40044041` | Default magic key for MPEG decode (hex) |
| `magicfhack` | `0` | Debug only: force a static f_code override (1-7), bypassing delta/delta |
| `audiolevel` | `100` | MPEG audio volume level (percentage) |
| `a204debug` | — | Heavy-debug build only: log function Ah/204h calls |
| `a206debug` | — | Heavy-debug build only: log function Ah/206h calls |

Example for Return to Zork with DUP5 and MIDI:
```ini
[reelmagic]
enabled=true
alwaysresident=true
vgadup5hack=true
audiolevel=100

[sdl]
fullresolution=desktop
output=openglnb

[render]
scaler=none
aspect=true

[midi]
mpu401=intelligent
mididevice=coremidi
midiconfig=0
```


---


# Architecture

```
                             |-----------------|                                    |---------------|
                             |     PL_MPEG     |                                    |   Existing    |
                             | Decoder Library |                                    |    DOSBox     |
                             |-----------------|                                    |   "Render"    |
                                      ^                                             |---------------|
                                      |                        |-------------|              ^
                                      | Uses                   |             |              | Mixed VGA + MPEG
                                      |           Outputs      |   DOSBox    |              | Output Goes Here
|---------------|              |-------------|    Audio to     |    Mixer    |      |---------------|
|               |              |             | --------------->|             |      |               |
|   Driver +    |              |    MPEG     |                 |-------------|      |     Video     |
|   Hardware    | <----------> |  Player(s)  |                                      |    Underlay   |
|   Emulation   |   Controls   |             | -----------------------------------> |     Mixer     |
|               |              |-------------|               Outputs                |               |
|---------------|                                            Video to               |---------------|
        ^                                                                                   ^
        | Can                                                                               | Video Output
        | Use                                                                               | Intercepted By
|----------------|                                                                  |---------------|
|    Emulated    |                                                                  |   Existing    |
|  DOS Programs  |                                                                  |    DOSBox     |
|----------------|                                                                  |     VGA       |
                                                                                    |   Emulation   |
                                                                                    |---------------|
```

### Source File Map

**Modified DOSBox files:**

| File | Change |
|------|--------|
| `include/logging.h` | Added `REELMAGIC` logging type |
| `src/debug/debug_gui.cpp` | Added `REELMAGIC` logging type |
| `src/dosbox.cpp` | ReelMagic init hook-in and config section |
| `src/hardware/Makefile.am` | Declared ReelMagic source files |
| `src/hardware/vga_dac.cpp` | VGA output redirect to ReelMagic mixer |
| `src/hardware/vga_draw.cpp` | VGA output redirect to ReelMagic mixer |
| `src/hardware/vga_other.cpp` | VGA output redirect to ReelMagic mixer |
| `src/gui/sdlmain.cpp` | Integer-scale OpenGL viewport for HiDPI fullscreen |
| `configure.ac` | ARM64/aarch64 target CPU detection |

**New ReelMagic files:**

| File | Purpose |
|------|---------|
| `include/reelmagic.h` | Public header for all ReelMagic interfaces |
| `include/vga_reelmagic_override.h` | VGA → ReelMagic render redirect macros |
| `src/hardware/reelmagic_driver.cpp` | `RMDEV.SYS` + `FMPDRV.EXE` emulation (INT 2F + INT 80h+) |
| `src/hardware/reelmagic_player.cpp` | MPEG player: decode, delta/delta f_code, audio FIFO |
| `src/hardware/reelmagic_videomixer.cpp` | VGA + MPEG compositing with z-order and alpha |
| `src/hardware/reelmagic_pl_mpeg.cpp` | Modified PL_MPEG with picture header callback hook |
| `src/hardware/reelmagic_pl_mpeg.h` | Modified PL_MPEG header (decode_picture_header_callback) |


---


# Building

## macOS (Apple Silicon / ARM64)

```bash
brew install autoconf automake libtool sdl12-compat
cd dosbox-0.74-3
bash autogen.sh
./configure
make -j$(sysctl -n hw.ncpu)
# Binary: src/dosbox (Mach-O 64-bit executable arm64)
```

## macOS (Intel)

Same as above. `configure` auto-detects x86_64.

## Ubuntu 20.04+

```bash
sudo apt install build-essential autoconf automake-1.15 autotools-dev m4 libsdl1.2-dev
cd dosbox-0.74-3
./autogen.sh
./configure
make -j$(nproc)
```

## Windows 10 (MinGW cross-compile from Ubuntu)

See `ci/win32-cross/README.md`:
```bash
cd ci/win32-cross
./install_dosbox_sdl_deps.sh
./cross_compile_dosbox.sh
./make_dosbox_zip_package.sh
```

Also: https://www.dosbox.com/wiki/Building_DOSBox_with_MinGW

## Debug Builds

```bash
./configure --enable-debug          # Standard debug (breakpoints, logging)
./configure --enable-debug=heavy    # Heavy debug (ReelMagic API trace logging)
```


---


# Documentation

| File | Contents |
|------|----------|
| `NOTES_MPEG.md` | Deep dive into the "magical" MPEG-1 format, f_code corruption, delta/delta recovery algorithm, and magic key analysis |
| `RMDOS_API.md` | Reverse-engineered ReelMagic DOS API: RMDEV.SYS INT 2F functions, FMPDRV.EXE driver_call() INT 80h+ API, streaming callbacks |
| `RMPORTIO_NOTES.md` | Physical ReelMagic board port I/O protocol, PL-450 DRAM/IMEM firmware loading |
| `KNOWN_GAME_BUGS.md` | Bugs verified on real hardware (not emulator defects) |
| `IMPROVEMENTS.md` | Analysis and recommendations for display, video, and audio quality |
| `example.dosbox.conf` | Example configuration file |


---


# Online Resources

## ReelMagic Hardware & Software

* https://www.vogons.org/viewtopic.php?t=7364
* https://www.vogons.org/viewtopic.php?t=37363
* http://bitsavers.informatik.uni-stuttgart.de/pdf/c-cube/90-1450-101_CL450_MPEG_Video_Decoder_Users_Manual_1994.pdf
* http://www.oldlinux.org/Linux.old/docs/interrupts/int-html/rb-5094.htm
* http://www.oldlinux.org/Linux.old/docs/interrupts/int-html/rb-4251.htm

## PL_MPEG Decoder Library

* https://phoboslab.org/log/2019/06/pl-mpeg-single-file-library
* https://github.com/phoboslab/pl_mpeg

## ReelMagic Debug Tools

* Voxam (GUI magical MPEG analyzer): https://github.com/jrdennisoss/voxam
* rmtools (CLI tools): https://github.com/jrdennisoss/rmtools


