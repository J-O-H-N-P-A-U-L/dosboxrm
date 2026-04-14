# Return to Zork ReelMagic — Crystal-Clear Resolution & Audio Recommendations

After a thorough analysis of the dosboxrm codebase, the game installation, display hardware (5K LG UltraFine @ 5120x2880), and MIDI setup (Roland MT-32 on CoreMIDI destination 0), here are findings and recommendations organized by priority.

---

## A. IMMEDIATE CONFIG FIXES (DOSBoxRTZ-RM.conf)

### 1. Output Mode: Use `openglnb` (Nearest-Neighbor) — Already Correct

The config already has `output=openglnb`. This is **correct** for pixel-perfect clarity. `opengl` (bilinear) would blur pixel art. Keep it.

### 2. Fix `fullresolution` — Change from `original` to `desktop`

```ini
# CURRENT (wrong for sharp scaling):
fullresolution=original

# RECOMMENDED:
fullresolution=desktop
```

**Why:** With `original`, SDL1 creates a fullscreen surface at the game's native resolution (320x240 after DUP5), then the monitor upscales from that tiny signal — the monitor's own scaler introduces blur. With `desktop`, the OpenGL viewport fills 5120x2880 and the `openglnb` nearest-neighbor filtering does the upscale GPU-side with perfect pixel sharpness.

### 3. Set `windowresolution` for Windowed Testing

```ini
windowresolution=1280x960
```

This gives a clean 4x integer scale of 320x240 for windowed debugging.

### 4. Fix `aspect` — Enable It

```ini
# CURRENT:
aspect=false

# RECOMMENDED:
aspect=true
```

**Why:** Return to Zork runs in VGA Mode 13h (320x200). The DUP5 hack converts this to 320x240 (4:3), which is correct. But if the VGA mode ever switches to non-DUP5 paths (menus, transitions), `aspect=true` ensures the DOSBox render pipeline applies proper 4:3 aspect ratio correction so you don't get crushed/stretched output.

### 5. Fix MIDI — Set `midiconfig=0` Explicitly

```ini
# CURRENT:
midiconfig=

# RECOMMENDED:
midiconfig=0
```

The Roland MT-32 is CoreMIDI destination 0 (confirmed — it's the only MIDI destination on this system). The empty `midiconfig=` may cause the `atoi()` fallback to work (returning 0), but setting it explicitly is correct practice.

### 6. Scaler Choice — `normal3x` vs `none`

```ini
# CURRENT:
scaler=normal3x

# RECOMMENDED for fullscreen:
scaler=none
```

**Why:** With `output=openglnb` + `fullresolution=desktop`, the OpenGL viewport handles all scaling to native display resolution. The DOSBox software scaler runs *before* the GL texture upload. `normal3x` triples the resolution to 960x720, then GL scales again to 5120x2880 — this doesn't divide evenly (5120/960 = 5.33x) causing **uneven pixel sizes**. With `scaler=none`, the GL texture is 320x240, and the viewport scales to fit the display. At 5120x2880, that's 16x horizontal and 12x vertical — both integers! **Perfect pixel grid.**

Calculation: 5120/320 = 16, 2880/240 = 12. Both integers. This is the **ideal** case for pixel-perfect rendering. Use `scaler=none` and let OpenGL handle it.

---

## B. RECOMMENDED OPTIMIZED CONFIG

Complete recommended `[sdl]`, `[render]`, `[midi]`, and `[reelmagic]` sections:

```ini
[sdl]
fullscreen=true
fulldouble=true
fullresolution=desktop
windowresolution=1280x960
output=openglnb
autolock=true
sensitivity=100
waitonerror=true
priority=higher,normal
mapperfile=mapper-0.74-3rm.map
usescancodes=true

[render]
frameskip=0
aspect=true
scaler=none

[midi]
mpu401=intelligent
mididevice=coremidi
midiconfig=0

[reelmagic]
enabled=true
alwaysresident=true
vgadup5hack=true
audiolevel=100
audiofifosize=30
audiofifodispose=2
```

---

## C. CODE-LEVEL IMPROVEMENTS (Repo Changes)

### C1. Bug Fix: `_mpegPictureWidth` vs `_mpegPictureHeight` in SetupVideoMixer

In `reelmagic_videomixer.cpp`, the MPEG-dictates-output-size path has what appears to be a copy-paste bug:

```cpp
if (_mpegDictatesOutputSize && mpeg) {
    _renderWidth = _mpegPictureWidth;
    _renderHeight = _mpegPictureWidth;  // BUG: should be _mpegPictureHeight
}
```

This code path isn't currently reachable (it would `E_Exit` below), but when eventually implemented this will cause square-stretched output. Fix it now.

### C2. ARM64 CPU Target — Add `aarch64` to configure.ac

This is the **highest-impact code change**. Currently ARM64 falls into the `*) UNKNOWN` case, which means:
- No dynamic recompiler core (`C_DYNREC` not defined)
- `c_unalignedmemory=no` (ARM64 supports unaligned access fine)
- Falls back to `core=simple` even when `core=auto` is set

In `configure.ac` around line 248, add an `aarch64` case:

```
  aarch64 | arm64)
    AC_DEFINE(C_TARGETCPU,ARMV8LE)
    AC_MSG_RESULT(ARM 64-bit)
    c_targetcpu="aarch64"
    c_unalignedmemory=yes
    ;;
```

And in the dynrec section (~line 341), add:
```
    if test x$c_targetcpu = xaarch64 ; then
        AC_DEFINE(C_DYNREC,1)
        AC_MSG_RESULT(yes)
    else
```

**Note:** DOSBox SVN trunk and DOSBox-X already have ARM64 dynrec support. The dynamic recompiler (`core_dynrec`) has architecture-specific backends. You would need to verify if the `core_dynrec/` directory has ARM64 codegen support (risc_armv8le.h or similar). If not, this is a larger porting effort — but even just setting `c_unalignedmemory=yes` for ARM64 is a worthwhile fix.

### C3. MPEG YCbCr → RGB Conversion Quality

The current pipeline in `reelmagic_player.cpp` calls:
```cpp
plm_frame_to_rgb(_nextFrame, (uint8_t*)outputBuffer, _attrs.PictureSize.Width * 3);
```

The embedded pl_mpeg library does YCbCr→RGB conversion using fixed-point integer math with limited precision. For higher-quality MPEG video output, consider:
- Using `plm_frame_to_rgba()` directly to avoid an extra format conversion step in the mixer
- Or better yet, doing the YCbCr→RGB conversion with proper BT.601 coefficients and clamping — the current pl_mpeg implementation can produce off-by-one color banding artifacts

### C4. OpenGL Integer Scaling (Letterboxing)

The current OpenGL path maps the quad to the full `-1..1` normalized viewport, meaning if the aspect ratio doesn't match, the display stretches. For the 5120x2880 (16:9) display showing 320x240 (4:3) content:

- 320x240 at 4:3 on a 16:9 display should be letterboxed (black bars on sides)
- The current code with `fullresolution=desktop` will stretch to fill

**Recommendation:** Modify `GFX_SetSize()` in `sdlmain.cpp` to compute a proper letterboxed viewport when `aspect=true`:

```cpp
// In the SCREEN_OPENGL case, after GFX_SetupSurfaceScaled:
if (sdl.desktop.fullscreen && aspect_correction) {
    // Compute the largest integer-scale rectangle that fits
    int scaleX = sdl.surface->w / width;
    int scaleY = sdl.surface->h / height;
    int scale = (scaleX < scaleY) ? scaleX : scaleY;
    int vw = width * scale;
    int vh = height * scale;
    glViewport((sdl.surface->w - vw) / 2, (sdl.surface->h - vh) / 2, vw, vh);
}
```

For the 5K display: `min(5120/320, 2880/240) = min(16, 12) = 12`. So viewport = 3840x2880, centered with 640px black bars on each side. Every source pixel maps to exactly a 12x12 block of display pixels. **Pixel-perfect.**

### C5. VGA DUP5 Hack Interpolation Quality

Currently the DUP5 hack literally duplicates every 5th scanline:
```
Line 1 → output line 1
Line 2 → output line 2
Line 3 → output line 3
Line 4 → output line 4
Line 5 → output line 5
Line 5 → output line 6 (duplicate!)
```

This creates a visible "thick line" artifact every 6 output lines. A better approach would be to distribute the extra lines more evenly using a Bresenham-style pattern:
```
5 source lines → 6 output lines, duplicate pattern: 1,1,1,2,1,1 (duplicate line 3 or 4)
```

Or, for the cleanest approach, don't use DUP5 at all — instead let the MPEG decoder output at its native resolution (typically 320x240 already) and adjust the VGA→MPEG size mapping. The MPEG videos from ZORK_MPEG.ISO are already 320x240 4:3; the VGA overlay is 320x200. The DUP5 hack stretches VGA to match. An alternative is to render at 320x200 and use the general-resize path to scale the MPEG from 240 to 200 lines, then let `aspect=true` handle the final 200→240 correction. This avoids the duplicated-line artifact entirely.

---

## D. BUILD INSTRUCTIONS FOR M4 MAC

There is currently **no native ARM64 build** — the existing binary is a Win32 PE32 executable. To build natively:

```bash
# Install dependencies
brew install autoconf automake libtool sdl12-compat sdl_net libpng

# Build
cd /Users/roland/Development/dosboxrm/dosbox-0.74-3
./autogen.sh
./configure --enable-core-inline --disable-dynamic-x86
make -j$(sysctl -n hw.ncpu)

# The binary will be at src/dosbox
```

**Key notes:**
- `sdl12-compat` provides SDL1.2 API compatibility on modern macOS
- `--disable-dynamic-x86` prevents trying to use x86 dynamic core on ARM
- The M4 is fast enough that `core=normal` with generous cycles will be fine for a 486-era game
- `cycles=max` or `cycles=fixed 20000` is likely optimal for RTZ

---

## E. AUDIO/MIDI SPECIFICS

### Roland MT-32 Setup

The config is almost correct. The game's `RTZ.BAT` already passes `-M:Drivers\MT32MPU` to MADERM.EXE, which routes MIDI through the MPU-401 interface. The DOSBox chain is:

```
Game → MPU-401 (intelligent mode) → DOSBox MIDI handler → CoreMIDI → Roland MT-32
```

The `midiconfig=0` targets the only CoreMIDI destination. This should work out of the box.

### MPEG Audio Level

The config comments out `audiolevel=150` (defaults to 150%). On a real ReelMagic card, the MPEG audio was mixed at a specific level relative to SB16/OPL. With a real MT-32 producing analog audio, the MPEG audio level may need lowering to balance:

```ini
audiolevel=100
```

Adjust to taste — the MT-32's analog output level vs. the DOSBox-mixed digital MPEG audio may need balancing.

---

## F. SUMMARY OF PRIORITY ACTIONS

| Priority | Action | Impact |
|----------|--------|--------|
| **1** | Set `fullresolution=desktop` | Eliminates monitor-scaler blur |
| **2** | Set `scaler=none` | Perfect 16x/12x integer scaling on 5K display |
| **3** | Set `aspect=true` | Correct 4:3 aspect ratio |
| **4** | Set `midiconfig=0` | Explicit MT-32 routing |
| **5** | Build native ARM64 binary | ~2-3x performance vs. Rosetta |
| **6** | Implement integer-scale viewport (C4) | True pixel-perfect letterboxing |
| **7** | Fix `_mpegPictureHeight` bug (C1) | Future-proofing |
| **8** | Add ARM64 to configure.ac (C2) | Native CPU target recognition |

Priorities 1-4 are config-only changes that can be made right now. The 5K LG UltraFine at 5120x2880 is mathematically perfect for this game: 320x240 × 12 = 3840x2880, with symmetric 640px letterbox bars. With `openglnb` + `scaler=none` + `fullresolution=desktop`, every source pixel maps to an exact 12×12 block of physical pixels — as sharp as it gets.
