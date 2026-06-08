# WORK.md — Exhaustive work log & resume guide

Detailed record of work done on `lr-yabasanshiro` (Sega Saturn libretro core).
Start with [`AGENTS.md`](AGENTS.md) for orientation and build/test commands.
This file is the deep log: what changed, why, where, how it was verified, and
what is still open.

Cross-session memory lives in
`/home/romain/.claude/projects/-home-romain-yaba2026-libretro/memory/`
(`MEMORY.md` is the index). This document links to those notes by name.

---

## 1. Repository setup (memory: `yaba-github-repo`)

- Initialized git, pushed to **github.com/rtissera/lr-yabasanshiro** (public,
  `main`, remote `origin`). Created fresh on GitHub (no native "forked from"
  badge); README states the fork relationship.
- **License:** GPL-2.0-or-later. `LICENSE` (GPL-2.0 text) shipped; matches
  per-file headers (inherited from Yabause/YabaSanshiro).
- **README.md:** marks fork of devmiyax/yabause (YabaSanshiro), base Yabause;
  documents Vulkan/GL/GLES builds and the feature additions.
- **Vendoring:** `libretro-common` and `musashi` vendored inline (libretro
  convention is vendoring, not submodules — geolith-libretro confirms). Stripped
  a dangling nested `.git` gitlink from libretro-common.
- **`.gitignore`:** excludes build artifacts (`*.o *.d *.so *.a`), generated
  tool exes, logs, `SS-*.png`, and ~631 MB of unused prebuilt Windows/Mac
  `*.lib/*.dll/*.dylib` (deliberate — keep them OUT).
- **`.gitattributes`:** `* text=auto eol=lf`; `libretro-common` + `musashi`
  marked `linguist-vendored`; m68kops `linguist-generated`.

### Critical vendoring gotcha (fixed, commit `12a09f0`)
The vendored `src/libretro/libretro-common/` shipped its **own** nested
`.gitignore` containing a `glsm/` rule. That hid `glsm/glsm.c` +
`include/glsm/glsm.h` → never committed → fresh clone / CI failed with
`src/ygl.h:50: fatal error: glsm/glsm.h: No such file`. Fix: deleted the nested
`.gitignore`, tracked glsm. **Lesson:** after vendoring a repo, run
`git status --ignored` to catch nested `.gitignore`s dropping needed source.

### `.git` bloat fixes
- Embedded git repo added as a gitlink → removed nested `.git`, re-added as files.
- `.git` ballooned to 123 MB from loose `.lib` objects →
  `git reflog expire --expire=now --all && git gc --prune=now` → 26 MB.

---

## 2. Core metadata & CI (memory: `yaba-github-repo`)

- **`yabasanshiro_libretro.info`:** `disk_control=true`,
  `memory_descriptors=true`, `savestate="true"`,
  `required_hw_api="Vulkan >= 1.0 | OpenGL >= 3.3 | OpenGL ES >= 3.0"`,
  extensions `bin|ccd|chd|cue|iso|mds|m3u|zip`, `display_version="v2.5.1"`.
- **`.github/workflows/build.yml`:** matrix `arch: [x86_64, aarch64]` ×
  `name: [vulkan, opengl, gles]`. Single `include` list maps runner per arch
  (`ubuntu-latest` / `ubuntu-24.04-arm` native runners) and make flags per name.
  Installs `build-essential nasm` + GL/GLES/Vulkan/shaderc dev libs, builds,
  uploads artifacts. **All 6 jobs green.**
  - YAML gotcha fixed: had two `include:` keys (invalid) → merged into one list.

---

## 3. Core options v2 (memory: `yaba-github-repo`)

- **`src/libretro/libretro_core_options.h`** (new): standard v2 helper.
  Categories system/video/input; 13 options with value labels, sublabels,
  defaults matching the legacy first-value. `libretro_set_core_options()`
  negotiates `GET_CORE_OPTIONS_VERSION` and auto-downgrades v2→v1→legacy
  `SET_VARIABLES`. English-only (`options_intl` NULL for other languages).
- **`src/libretro/libretro.c`:** `#include "libretro_core_options.h"`; in
  `retro_set_environment` replaced the old `vars[]` + `SET_VARIABLES` with
  `libretro_set_core_options(cb, &option_cats_supported)`. Keys/defaults
  unchanged → existing `.opt` files still load.
- **Minimum frontend:** core options v2 = RetroArch **1.10.0+** (older
  frontends fall back automatically).

### OPTION_DISPLAY callback (commit `3bd1c93`)
`update_core_options_display()` + `core_options_update_display_cb()`
(HAVE_VULKAN-gated) hide the upscale knobs (`yabasanshiro_resolution_mode`,
`rbg_resolution_mode`, `rbg_use_compute_shader`, `polygon_mode`) when
`video_core=software`, via `RETRO_ENVIRONMENT_SET_CORE_OPTIONS_DISPLAY`.
**Gotcha:** the macro is `SET_CORE_OPTIONS_DISPLAY` (plural OPTIONS), not the
singular `SET_CORE_OPTION_DISPLAY`. Registered through
`SET_CORE_OPTIONS_UPDATE_DISPLAY_CALLBACK`.

---

## 4. Renderer fixes — Vulkan (memory: `yaba-vulkan-remaining-black-output`, `yaba-libretro-features`)

### 4a. Black screen root cause (resolved)
`yabasanshiro_resolution_mode` mapped **every** choice to the upscale path,
ending in `VIDVulkan::blitSubRenderTarget()`, which is broken under the libretro
HW context (layout mismatch: leaves `_out_img` in `GENERAL` while `set_image`
declares `SHADER_READ_ONLY_OPTIMAL`). Fix: map "original" → `g_resolution_mode=0`
(RES_NATIVE) in **both** `retro_set_resolution()` and `check_variables()` in
`libretro.c` (the switch in `retro_set_resolution` was clobbering
`check_variables`). RES_NATIVE renders straight into the window framebuffer, no
blit. Verified via readback: nonblack 67766/76800.

### 4b. Upscale 2x/4x black (commit `e122a37`) — three bugs, all driver-independent
Found via the Vulkan validation layer (Steam's
`libVkLayer_khronos_validation.so` + a hand-written manifest at
`/tmp/vklayer/validation.json`):

1. **Image usage flags.** `_out_img` (`vidvulkan_bridge.cpp`) lacked
   `TRANSFER_DST` (blit dest) + `SAMPLED` (RP finalLayout SHADER_READ_ONLY +
   frontend samples it); `subRenderTarget.color.image` lacked `TRANSFER_SRC`
   (blit source). Now `_out_img` =
   `COLOR_ATTACHMENT | TRANSFER_SRC | TRANSFER_DST | SAMPLED`.
2. **Render-pass incompatibility** (VUID-…-02684): subRenderTarget RP had
   `dependencyCount=2` but the window RP the VDP2 pipelines are built against has
   `0` → every draw rejected. Set `dependencyCount=0` under `__LIBRETRO__`
   (`VIDVulkan.cpp generateSubRenderTarget`).
3. **The killer.** After `blitSubRenderTarget` the upscale path began the "keep"
   render pass (`loadOp=CLEAR`) on `_out_img` and ended it (OSD-only, disabled on
   libretro) → wiped the blit to black. Wrapped that begin/end in
   `#ifndef __LIBRETRO__` (`VIDVulkan.cpp` ~line 1187).

Verified via `YABA_READBACK`: native ~95%, 2x ~94%, 4x ~94% nonblack.

### 4c. GetNativeResolution + offscreen crash (commit `e8c576f`)
- `VIDVulkan::GetNativeResolution()` (~line 1610) was a no-op stub; now reports
  `*width=vdp2width; *height=vdp2height; *interlace=_vdp2_interlace;`
  (`_vdp2_interlace` is the member — `vdp2_interlace` is undeclared).
- `YuiSwapBuffers()` calls GetNativeResolution; on change → `retro_set_resolution()`.
- **Crash root cause + fix:** `generateOffscreenPath()` called
  `deleteOfscreenPath()` (nulls `offscreenPass.frameBuffer`) **before** the
  dims-unchanged early return → a same-size rebuild freed the FB then returned
  without recreating → next `renderToOffecreenTarget` did `vkCmdBeginRenderPass`
  on a NULL framebuffer → SIGSEGV. Fix: compute `wantW/wantH`, decide the skip
  (including FB validity: `offscreenPass.frameBuffer != VK_NULL_HANDLE` and dims
  match) **before** deleting; then delete + recreate unconditionally (removed the
  inner early-returns that left a NULL framebuffer).
- Made the offscreen path libretro-compatible: `offColorFormat` = B8G8R8A8 under
  `__LIBRETRO__` (R8G8B8A8 else); subpass `pDepthStencilAttachment=nullptr`; RP
  `attachmentCount=1`/`dependencyCount=0`; FB `attachmentCount=1`.
- Verified: 2700+ frames, geometry tracked 320x224 → 320x240 → 640x480, no crash.

### 4d. Aspect / "truncation" (memory: `yaba-libretro-features`)
The on-screen "truncated to a corner" was **RetroArch config**, not the core:
`aspect_ratio_index="23"` (Custom) + a fixed `custom_viewport_*`. Fix: set
Aspect Ratio → **Core Provided**. Kept the related code improvement: under
`__LIBRETRO__`, `SetSaturnResolution()` aspect-centering is bypassed (`if(false)`)
so origin stays 0,0 and render fills `deviceWidth×deviceHeight` — the frontend
owns aspect via `geometry.aspect_ratio`. Also fixed a `resize()` self-assign
(`originx = originx` → `this->originx = originx`).

### 4e. Clean-quit crash (memory: `yaba-libretro-features`)
`retro_unload_game` had `if (!renderer_running) VIDCore->Init();` — on clean quit
`context_destroy` ran first (DeInit'd VIDCore + nulled the HW interface), then
unload re-Init'd with no valid Vulkan context → crash. Fix (`libretro.c`): unload
does `if (!renderer_running) VIDCore = NULL;` (detach so
`YabauseDeInit→VideoDeInit` skips a double-DeInit); `context_destroy` null-guards
`if (renderer_running && VIDCore) VIDCore->DeInit();`. Handles both teardown
orders and the resize destroy/reset cycle.

---

## 5. Renderer — OpenGL / GLES (memory: `yaba-libretro-features`)

- **Desktop GL** (VIDOGL, `video_core=opengl`) WORKS — verified headless on
  llvmpipe GL 4.5 (nonblack 0.934).
- **GLES build** (`make FORCE_GLES=1 HAVE_VULKAN=0`) was broken; fixed to
  compile + link:
  - Makefile: added `-DHAVE_GLSYM_PRIVATE` to the FORCE_GLES branch.
  - `src/libretro/glsym_private.h`: added 4 missing GLES 3.1 compute enums
    (`GL_MAX_COMPUTE_WORK_GROUP_INVOCATIONS/COUNT/SIZE`, `GL_ALL_BARRIER_BITS`).
  - Must `make clean` when switching GL↔GLES.
- **GLES link bug** (commit `70ab055`): arch blocks hardcoded `-lGL` which
  preceded `-lGLESv2` and won via `--as-needed`, so FORCE_GLES linked libGL.
  Moved the GL lib to follow renderer choice: `-lGLESv2` for FORCE_GLES, `-lGL`
  else. Verified gles NEEDs `libGLESv2.so.2` only.
- **GLES runtime untested locally** — the local RetroArch is compiled GL-only
  ("Requesting OpenGLES3 context, but RetroArch is compiled against OpenGL").
  Needs a GLES-enabled RetroArch (ARM/mobile/RetroPie). GLES `.so` kept at
  `/tmp/yaba_gles.so`.

---

## 6. Savestates (memory: `yaba-savestates`)

Were stubbed off ("Disabling savestates until they are safe"). Re-enabled + fixed:

1. libretro requires `retro_serialize_size()` **constant** for the loaded game,
   but Yabause's raw state size varies frame-to-frame → load mismatch. Fix
   (`libretro.c`): cache an upper bound on first probe
   `g_serialize_size = size + size/4 + 2MB`, return it for the session; reset to
   0 in `retro_load_game`; `retro_serialize` zero-pads to fill.
2. Padding then tripped `YabLoadStateStream`'s exact-size check
   (`src/memory.c:1795`). Relaxed `!=` → `(ftell-headersize) < size` (the
   embedded header size is authoritative; chunks self-describe; pad never read).

Verified headless: `serialize_size` constant, ~1 MB compressed `.state`, load
succeeds and reverts to the saved scene. **Frontend gotcha:** a stale
`~/.config/retroarch/cores/core_info.cache` can mark the core
savestate-unsupported → refresh/delete it (or
`core_info_cache_enable=false`).

---

## 7. Feature additions (memory: `yaba-libretro-features`)

1. **Disk control** (multi-disc / `.m3u`) — ahead of upstream. Wires
   `SET_DISK_CONTROL_EXT_INTERFACE` (falls back to legacy) to `cs2.c`: eject =
   `Cs2ForceOpenTray()`, insert = `Cs2ForceCloseTray(CDCORE_ISO, path)`.
   `disk_init_from_content()` parses `.m3u` (one disc/line, `#` comments) or a
   single image; honours set_initial_image.
2. **SYSTEM_RAM** — `retro_get_memory_data/size(RETRO_MEMORY_SYSTEM_RAM)` =
   HighWram (0x06000000, 0x100000). Size returned unconditionally (HighWram is
   allocated late in YabauseInit, after the frontend queries size).
3. **SAVE_RAM** via a shadow buffer (avoids a NULL-crash): static
   `sram_shadow[0x10000]`, always valid; `sram_seed_from_backup()` migrates an
   existing `backup.bin`; `sram_sync()` pushes shadow→BupRam once then mirrors
   BupRam→shadow each frame. Frontend overlays `.srm`. Verified `.srm` written
   65536 and reloads.
4. **SET_MEMORY_MAPS** (full RA): `set_memory_maps()` from `context_reset()`
   after YabauseInit. Two `RETRO_MEMDESC_SYSTEM_RAM` descriptors — HWRAM ptr
   HighWram start 0x06000000 len 0x100000; LWRAM ptr LowWram start 0x00200000
   len 0x100000.

**Init-order gotcha:** `YabauseInit()` (allocates BupRam/HighWram) runs in
`context_reset()` (HW render callback), AFTER `retro_load_game` returns — so
anything the frontend queries at load time (memory sizes, savestate size) must
not depend on those buffers existing yet.

---

## 8. RetroAchievements (commit `ac246d3`, memory: `yaba-github-repo`)

RA supports Saturn (console id 39, ~184 sets). The **exact memory convention**
(from rcheevos #302, Beetle `mednafen-endian.h`, rcheevos `consoleinfo.c`
`_rc_memory_regions_saturn`): WorkRAM Low/High are exposed **16-bit
byte-swapped on a little-endian host** — `RA_offset = SH2_offset ^ 1`. Not
native-BE, not 32-bit word-swap. rcheevos reads raw bytes and **ignores**
`RETRO_MEMDESC_BIGENDIAN`. There is **no core-whitelist hard-block** in rcheevos
(only per-core disallowed *settings*), so no frontend hook is needed or possible.

**Key finding: our buffers already match.** HighWram/LowWram use the T2 macros =
`mem[addr ^ 1]` on LE (`src/memory.h:108-152`), and `set_memory_maps()` exposes
them at 0x00200000 (LWRAM) / 0x06000000 (HWRAM) — exactly rcheevos' Saturn
regions. No de-swizzle shadow, no frontend patching. Cross-checked vs the #302
Dark Savior example (SH2 0xFE77C → RA 0x1FE77D) — byte-identical to Beetle.

**Change:** added `RETRO_ENVIRONMENT_SET_SUPPORT_ACHIEVEMENTS` (true) in
`retro_set_environment`. Verified RetroArch logs `SET_SUPPORT_ACHIEVEMENTS: yes`.

**Remaining (not code):** empirical validation needs an RA account + network
(login gates memory init — can't do headless here). Official "supported
core"/hardcore listing is a separate RA-team submission.

---

## 9. SH2 dynarec x86_64 (memory: `yaba-dynarec-x86_64`, `yaba-dynarec-optest-differential`, `yaba-arm-build-box`)

The devmiyax SH2 JIT (`src/sh2_dynarec_devmiyax/`) was never functional on Linux
x86_64. It now **boots, runs the game, produces audio, no crash** — but VDP2
display-on (TVMD) never sets, so the screen stays black. **Use the interpreter**
(verified working).

Codegen is proven clean: a standalone GoogleTest opcode differential harness
(`src/sh2_dynarec_devmiyax/optest/`) against the known-good aarch64 dynarec
(oracle on `gx10-a3a1.local`) shows the x86_64 fail set == ARM fail set exactly
(real-build config: `CACHE_ENABLE=1` + `use_sh2_cache=1`). Fixes applied (all in
source): x86_64 fragment-size macros, MS-x64 ABI on dynaFunc + helpers, opdesc
template sizes, `overrideMemFunc` full-8-byte immediate match (was patching only
low 32 bits → dead under ASLR), DIV1 opdesc `389,6,6`→`389,62,6`.

**Open bug:** master-ISR-vs-main interrupt/timing interleaving, x86-specific
(ARM dynarec runs the game). First x86-vs-interpreter divergence at PC 0x060108b6.
Cycle counting is correct and identical x86/ARM in isolation; the bug only appears
in the full system. See `PROGRESS_DYNAREC.md` and the dynarec memory files for the
detailed next-step options.

---

## 10. Current state & open issues (2026-06-08)

**Working:** Vulkan native + 2x + 4x render; desktop GL renders; savestates;
disk control; SAVE_RAM; memory maps; RetroAchievements hint. All 6 CI jobs green.

Recommended config for a clean picture: `video_core=vulkan` (or `opengl`),
`sh2coretype=interpreter`, `frameskip=disabled`, RetroArch aspect = Core Provided.

**Still open:**
- Intermittent SIGSEGV in `libvulkan_radeon` during the **frontend present**
  (~every few hundred frames); HW readback / SCREENSHOT also crash. Likely
  `set_image` needs a completion semaphore (currently sync via
  `vkDeviceWaitIdle` only).
- **SH2 dynarec** timing bug (§9) → black; use interpreter.
- **Auto-frameskip** skips all frames after frame 0 → use `frameskip=disabled`.
- **VDP1 sprite color noise** in Taroumaru intro (native res): arcs of
  random-colored single pixels around small sprites — suspect VDP1 sprite
  palette / color-bank / end-code (transparent) handling. Plus a dark
  horizontal bar near top. (The apparent "striping" is real artwork, not
  interlace — autocorrelation found no period.)
- **GLES runtime untested** — needs a GLES-enabled RetroArch.

**Recent commits:** `ac246d3` (RetroAchievements), `e8c576f` (GetNativeResolution
+ offscreen crash), `e122a37` (upscale 2x/4x black), `3bd1c93` (option display +
aarch64 CI), `3ca4dac` (core options v2), `12a09f0` (track glsm), `70ab055`
(.info/.gitattributes/CI + GLES link), `d809a98` (README + LICENSE), `e12b85b`
(initial).

---

## How to resume

1. Read [`AGENTS.md`](AGENTS.md) for build/test commands and the headless-only
   constraint.
2. Read the relevant memory file(s) for the area you're touching (table in
   AGENTS.md).
3. Build + verify headless with the `YABA_READBACK` probe — **do not eyeball
   screenshots**.
4. When you learn something durable, update the matching memory file (and its
   one-line pointer in `MEMORY.md`), not just this doc.
