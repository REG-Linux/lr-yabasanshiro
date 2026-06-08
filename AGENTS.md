# AGENTS.md — Orientation for AI agents working on this repo

This file is the **entry point** for any agent (Claude Code or other) resuming
work on the `lr-yabasanshiro` libretro core. Read this first, then read
[`WORK.md`](WORK.md) for the exhaustive change log and open issues.

## What this is

A [libretro](https://www.libretro.com/) core for the **Sega Saturn**, built from
**YabaSanshiro** (devMiyax's fork of Yabause). This repo is a fork carrying Linux
build fixes + extra libretro integration. License: **GPL-2.0-or-later**.

- Repo root: `/home/romain/yaba2026/libretro`
- GitHub: **github.com/rtissera/lr-yabasanshiro** (public, branch `main`, remote `origin`)
- Output artifact: `yabasanshiro_libretro.so`

## Project memory (read these)

Persistent cross-session notes live in
`/home/romain/.claude/projects/-home-romain-yaba2026-libretro/memory/`.
`MEMORY.md` there is the index. Key files:

| Memory file | Covers |
|---|---|
| `yaba-github-repo.md` | Git/GitHub setup, repo hygiene, core options v2, RetroAchievements, CI |
| `yaba-libretro-features.md` | Feature gaps vs upstream; disk control, SYSTEM_RAM, SAVE_RAM, memory maps, GL/GLES, clean-quit fix |
| `yaba-savestates.md` | Savestate re-enable + fixes (constant serialize size) |
| `yaba-vulkan-test-setup.md` | **How to build + headless-test the Vulkan core** |
| `yaba-vulkan-remaining-black-output.md` | Black-output root causes (resolved) + still-open Vulkan bugs |
| `yaba-dynarec-x86_64.md` | SH2 JIT x86_64 port — boots/runs/audio, one timing bug left (black) |
| `yaba-dynarec-optest-differential.md` | Standalone opcode differential harness (codegen proven clean) |
| `yaba-arm-build-box.md` | `gx10-a3a1.local` aarch64 oracle box for differential testing |

Update the relevant memory file (not this AGENTS.md) when you learn something
non-obvious that should persist across sessions.

## Build

```sh
make HAVE_VULKAN=1 -j$(nproc)        # Vulkan (default renderer)
make HAVE_VULKAN=0 -j$(nproc)        # Desktop OpenGL (VIDOGL)
make FORCE_GLES=1 HAVE_VULKAN=0 -j$(nproc)   # OpenGL ES 3.0/3.1
```

- `__LIBRETRO__` is defined for all translation units.
- **Switching renderers requires `make clean`** (mixing `_OGL3_`/`_OGLES3_`
  objects = link errors; switching to/from Vulkan needs `libretro.c.o` +
  `vidvulkan_libretro.c.o` rebuilt).
- Floors: GL 3.3 (4.3 for compute), GLES 3.0 (3.1 for compute), Vulkan 1.0.

## Headless testing (NO desktop pollution — hard constraint)

The user works headless and does **not** want windows opened on the real
display without explicit authorization. Two paths:

- **lavapipe (software Vulkan):** `VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/lvp_icd.json xvfb-run -a retroarch -L ./yabasanshiro_libretro.so <game.chd>`
- **RADV refuses Xvfb** ("No DRI3 support" → no graphics queue). RADV needs a
  real display; only use windowed-on-`:0` with explicit user OK, and capture
  ONLY the RetroArch window (`xdotool search --name RetroArch` + `import -window`),
  never the desktop root.

Pixel verification: **do not eyeball the X11 flip surface** (returns black).
Use the GPU readback probe — env var **`YABA_READBACK=1`** (one-shot
`vkCmdCopyImageToBuffer` at frame ~600, logs non-black pixel count). Note: the
older memory mentions `YABA_PROBE`; the current var is `YABA_READBACK`.

Xvfb gotcha: kill with `pkill -x Xvfb` — **NOT** `pkill -f "Xvfb :99"` (that
string is in your own shell cmdline → self-kill).

Assets on this box: BIOS `~/.config/retroarch/system/saturn_bios.bin`; games
`~/Bureau/REGLINUX/roms/saturn/*.chd`; core opts
`~/.config/retroarch/config/YabaSanshiro/YabaSanshiro.opt`.

## Known-good config for a working picture

- `yabasanshiro_video_core` = `vulkan` (or `opengl`)
- `yabasanshiro_sh2coretype` = **`interpreter`** (dynarec still has a timing bug → black)
- `yabasanshiro_frameskip` = **`disabled`** (auto-frameskip skips all frames after 0)
- `yabasanshiro_resolution_mode` = any (native/2x/4x all render now)
- RetroArch aspect ratio = **Core Provided** (else a stale custom viewport crops)

## Working style notes (from the user)

- **Caveman mode** is active in chat: terse responses, drop articles/filler.
  **Code, commits, PRs, and security warnings are written normally** — this
  applies to file contents like this one too.
- Commit/push only when asked. Branch off `main` is the working branch already.
- Don't open real-display windows without authorization. Headless only.
- Stop eyeballing screenshots — verify pixels with the readback probe.

## Current state (2026-06-08)

All 6 CI jobs green (x86_64 + aarch64 × vulkan/opengl/gles). Vulkan native + 2x +
4x render. Savestates, disk control, SAVE_RAM, memory maps, RetroAchievements
hint all landed. See [`WORK.md`](WORK.md) for the full picture and the open-issue
list (dynarec timing, intermittent RADV present crash, VDP1 sprite color noise,
GLES runtime untested).
