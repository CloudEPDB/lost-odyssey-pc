# Lost Odyssey — PC Port

**A native PC build of Lost Odyssey (Xbox 360), produced by static recompilation of the original game code. Not an emulator.**

*[Léeme en español](README.es.md)*

> **Status:** in development · playable · source not public yet · no downloads
> This repository is a progress window, not a release. See the [FAQ](docs/faq.md).

---

## What this is

Lost Odyssey shipped for the Xbox 360 in 2007 and never came to PC. This project turns it into a real PC executable.

Static recompilation translates the game's original PowerPC machine code, function by function, into x86-64 source code, which is then compiled into a normal Windows binary. There is no CPU emulation at runtime, no interpreter, no JIT: the game's logic runs as native code on your processor. Only the parts the game asks of the console — the GPU command stream, the filesystem, saves, audio, controller input — are reimplemented on the host.

The practical difference is that the game stops behaving like a console game running under emulation and starts behaving like a PC game. It can be modified. Its renderer can be changed. Its resolution is not fixed by what fits in a 2007 GPU's memory.

Built on the [ReXGlue](https://github.com/rexglue/rexglue-sdk) recompilation SDK (0.10.0), with a heavily modified fork of its Xenos GPU plugin.

---

## Current status

| | |
|---|---|
| **Boots and plays** | Yes — main menu, saves, achievements, cutscenes |
| **Renderers** | Direct3D 12 and Vulkan, both in one plugin, switchable in-game |
| **1080p native** | Working on D3D12 · in progress on Vulkan |
| **1440p / 4K** | Working on both |
| **Texture replacement** | Working on D3D12 · pending on Vulkan |
| **Linux** | Not built yet — the SDK supports it, including arm64 |
| **Android** | Not supported by the SDK |

Honest caveat: "playable" means it boots, runs, saves and has been played for extended sessions. It has not been verified start-to-finish across all four discs.

---

## What it adds over the Xbox 360 release

### Real 1080p, not upscaling

The 360 renders Lost Odyssey at 1280×720 because that is what fits in the console's 10 MB of EDRAM — colour and depth buffers together. The interesting part of this project is that the game now renders a genuine 1920×1080 frame, HUD and menus included, rather than a 720p frame stretched to fit your monitor.

Getting there required enlarging the emulated EDRAM eightfold, widening the render-target address fields, binary-patching the GPU plugin's precompiled resolve shaders, and rewriting the game's 2D projection per draw call so the interface follows the larger canvas. [How it works →](docs/technical.md)

Resolution options:

| Preset | Internal render | Notes |
|---|---|---|
| 720p | 1280×720 | Original console mode |
| **1080p (native)** | 1920×1080 | The game itself renders 1080p |
| 1440p | 2560×1440 | 2× supersample of the 720p canvas |
| 4K | 3840×2160 | 3× supersample of the 720p canvas |

On top of the preset: 1×/2×/3× SSAA, FXAA (off / normal / extreme), and a presentation filter (bilinear, CAS or FSR).

### An in-game options menu

Press **F2**. Resolution, antialiasing, renderer, patches, turbo and textures, in one place, in the game, without editing config files. Changes that can be applied live are applied live; those that cannot relaunch the game for you.

### Game patches, toggleable at runtime

The community patches from Xenia Canary (original patch work by **boma**) are reimplemented as recompiler hooks rather than byte patches, so each one is a switch you can flip while playing:

60 fps · character flicker fix · disable occlusion queries · post-process upscale fix · disable depth of field · disable motion blur · 16× anisotropic filtering · disable dynamic shadows

### Texture replacement

Dump every texture the game uses to PNG, replace what you want, and reload the pack in place with **F7** — no restart, no repacking.

### Turbo

Fast-forward at 1.5×, 2× or 3×, as hold or toggle, bindable to a controller button (**F6** on keyboard). Useful for a 2007 JRPG's random encounters and long corridors.

### A long-standing crash, fixed

The recompiled build originally died after roughly 27 minutes of play with a heap allocation failure. That is fixed.

---

## Screenshots

*Coming soon — see [media/](media/).*

---

## Documentation

- **[Technical notes](docs/technical.md)** — how 1080p native was actually achieved: EDRAM windowing, shader binary patching, canvas pinning, and the build trap that cost an evening.
- **[Progress log](docs/progress.md)** — what changed and when.
- **[FAQ](docs/faq.md)** — including where the source is and why.

---

## Legal

This repository contains **no game code, no game assets, and no executables** — only documentation and screenshots.

Lost Odyssey is © Microsoft / Mistwalker / Feelplus. This is an unaffiliated, non-commercial preservation and porting effort. Nothing here will ever distribute the game: any future release would require you to supply your own legally obtained copy.

Documentation in this repository is © its author. All rights reserved.
