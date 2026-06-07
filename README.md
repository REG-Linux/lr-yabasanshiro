# lr-yabasanshiro

A [libretro](https://www.libretro.com/) core for the Sega Saturn, built from
**YabaSanshiro** — Sega Saturn emulator by [devMiyax](https://github.com/devmiyax),
itself derived from the [Yabause](https://github.com/Yabause/yabause) project.

> **This is a fork.** Upstream: [devmiyax/yabause](https://github.com/devmiyax/yabause)
> (YabaSanshiro). This repository carries Linux build fixes and additional
> libretro integration on top of that work.

## What this fork adds

- **Vulkan** hardware renderer (`RETRO_HW_CONTEXT_VULKAN`) — default build
- **Desktop GL** and **OpenGL ES 3.0 / 3.1** builds wired and building cleanly
- Savestates re-enabled and fixed (stable serialize size, padded writes,
  relaxed load size check)
- Disk control interface (m3u / multi-disc swapping)
- `RETRO_MEMORY_SAVE_RAM` (`.srm` backup), `SYSTEM_RAM`, and memory maps
  (cheats / RetroAchievements)
- Clean-quit crash fix (renderer lifecycle on unload / context destroy)
- libretro full-frame resolution (no aspect-fit cropping in the core; let the
  frontend handle aspect)

## Building (Linux, x86_64 / aarch64)

```sh
# Vulkan (default)
make HAVE_VULKAN=1 -j$(nproc)

# Desktop OpenGL
make -j$(nproc)

# OpenGL ES 3.0 / 3.1
make clean && make FORCE_GLES=1 HAVE_VULKAN=0 -j$(nproc)
```

Output: `yabasanshiro_libretro.so`.

A BIOS is recommended for best compatibility; place it where your frontend
expects Saturn BIOS files (e.g. `saturn_bios.bin` in the RetroArch `system`
directory).

## Vendored dependencies

[`libretro-common`](https://github.com/libretro/libretro-common) is vendored
inline under `src/libretro/libretro-common`.

## License

GNU General Public License, version 2 or (at your option) any later version
(**GPL-2.0-or-later**) — inherited from Yabause / YabaSanshiro. See
[`LICENSE`](LICENSE).

Copyright holders include (non-exhaustive — see per-file headers):

- Guillaume Duhamel, Theo Berkau, Anders Montonen, and the Yabause contributors
- devMiyax (`smiyaxdev@gmail.com`) — YabaSanshiro

This fork's modifications are likewise distributed under GPL-2.0-or-later.
