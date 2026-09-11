# Lost Odyssey — PC Port

**A native PC build of Lost Odyssey (Xbox 360), produced by static recompilation of the original game code. Not an emulator.**

*[Léeme en español](README.es.md)*

![Lost Odyssey running on PC](media/hero.png)

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
| **1080p native** | Working on both |
| **1440p / 4K** | Working on both |
| **SMAA** | Working on both |
| **Texture replacement** | Working on both |
| **Settings in the game's own menu** | Working |
| **All four discs** | Disc changes handled automatically — extracted folders, ISO or Games on Demand |
| **Linux** | Not built yet — the SDK supports it, including arm64 |
| **Android** | Not supported by the SDK |

Honest caveat: "playable" means it boots, runs, saves and has been played for extended sessions. The disc change has been tested by forcing it, not yet at a real chapter boundary, and the game has not been verified start-to-finish across all four discs.

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

On top of the preset: 1×/2×/3× SSAA, post-process antialiasing (FXAA, FXAA extreme or SMAA 1x), and a presentation filter (bilinear, CAS or FSR).

### SMAA

Subpixel Morphological Antialiasing — the reference implementation, unmodified — running as three compute passes on the final frame, in both renderers. Cleaner edges than FXAA, without FXAA's softening of the whole image. [How it fits in →](docs/technical.md#5-smaa-on-the-final-frame)

TAA was considered and deliberately left out for now: done properly it needs per-shader camera jitter, the scene before the HUD, depth, history and motion vectors.

### Settings inside the game's own menu

Open the game's **Configuration** screen and press **RB**. Next to the original page there are four new tabs — **Graphics**, **Patches**, **Extras** and **Textures** — that look as if they shipped with the game, because they are drawn with the game's own font, metal panels and cursor.

Those assets are read at runtime from your own copy of the game. Nothing from the game is part of this project.

It works like the native page: up and down to move, left and right to change a value, **LB/RB** to switch tabs, **B** to go back to the game's own options. Changes that can apply immediately do. Those that need a restart are saved, and the page offers to restart the game — pressing A twice, so an accidental press never costs you unsaved progress. [How the tabs are built →](docs/technical.md#7-new-menu-pages-that-look-native)

The old **F2** overlay still exists during development and is on its way out.

### Game patches, toggleable at runtime

The community patches from Xenia Canary (original patch work by **boma**) are reimplemented as recompiler hooks rather than byte patches, so each one is a switch you can flip while playing:

60 fps · character flicker fix · disable occlusion queries · post-process upscale fix · disable depth of field · disable motion blur · 16× anisotropic filtering · disable dynamic shadows

### Save anywhere

An optional toggle that enables **Save** in the System menu away from save points. It uses the game's own save flow — the same slot screen, the same save files — instead of faking a save point.

Meant for exploration. The game was never designed to be saved in the middle of an event or a cutscene, so that is best avoided.

### Four discs, no swapping

Lost Odyssey spans four discs, and asks for the next one as the story moves on. On the 360 the console handles that. Here the port does: when the game asks for a disc, the port finds it, mounts it and lets the game continue. No prompt, no menu.

Discs are recognised by the header of their own executable ("disc N of 4"), so file and folder names do not matter. The intended layout is one folder per disc next to the executable:

```
Lost Odyssey\
├── lostodyssey.exe
└── data\
    ├── disc1\    default.xex, LO.fpi, xenon_*.fpd ...
    ├── disc2\
    ├── disc3\
    └── disc4\
```

Extracted folders are the recommended form, but **ISO images** and **Games on Demand** packages work as well, read where they are without extracting or copying anything, and so does pointing at a disc's `default.xex`. If the disc the game asks for cannot be found, the port says so and waits, as the console would, so it can be added without closing the game. [How disc changes work →](docs/technical.md#8-four-discs)

### Texture replacement

Dump every texture the game uses to PNG, replace what you want, and reload the pack in place with **F7** — no restart, no repacking.

Textures are matched by a hash of their contents rather than by memory address, so a pack keeps working across sessions and save files.

### DualSense button prompts

The game's button glyph atlas is one of those replaceable textures, so the on-screen prompts can show PlayStation glyphs instead of the Xbox ones the 2007 release hardcoded. No patching, no separate build — it ships as part of the texture pack.

### Turbo

Fast-forward at 1.5×, 2× or 3×, as hold or toggle, bindable to a controller button (**F6** on keyboard). Useful for a 2007 JRPG's random encounters and long corridors.

### A long-standing crash, fixed

The recompiled build originally died after roughly 27 minutes of play with a heap allocation failure. That is fixed.

---

## Screenshots

### 720p vs native 1080p

The same save, the same camera, two presets. Look at the interface, not the scenery: on the left it is drawn on the console's 1280×720 canvas and stretched to fit your screen. On the right the game is drawing it at 1920×1080.

| 720p — original console mode | 1080p — native |
|---|---|
| ![720p](media/comparison-720p.png) | ![1080p native](media/comparison-1080p.png) |

### Settings inside the game

The game's Configuration screen with the port's Graphics tab open. The font, the brushed-metal panels, the cursor and the layout are the game's own, read from its data at runtime; the page itself is new.

![Settings inside the game's own Configuration screen](media/in-game-settings.png)

### DualSense button prompts

The game's own settings screen, with PlayStation glyphs in place of the Xbox buttons the 2008 release hardcoded.

![DualSense glyphs](media/dualsense-glyphs.png)

### Title screen

The "HD Remaster" subtitle is not in the original game. It is a replaced texture — the texture pack at work on the very first thing you see — and it doubles as a way to tell at a glance which build you are running.

![Title screen](media/title-screen.png)

### The F2 overlay

The development overlay that came first. Now that the settings live in the game's own Configuration screen, it is being retired.

![F2 options overlay](media/options-menu.png)

---

## Documentation

- **[Technical notes](docs/technical.md)** — how 1080p native was actually achieved (EDRAM windowing, shader binary patching, canvas pinning), SMAA, new menu pages built from the game's own assets, and how the four discs are handled.
- **[Progress log](docs/progress.md)** — what changed and when.
- **[FAQ](docs/faq.md)** — including where the source is and why, and what you will need to play.

---

## Credits

Built and maintained by **[FaliGame](https://github.com/FaliGame)**.

Standing on other people's work:

- **[ReXGlue](https://github.com/rexglue/rexglue-sdk)** — the static recompilation SDK this port is built on, and the Xenos GPU plugin this project forks.
- **[Xenia](https://xenia.jp/)** — the emulator whose GPU research underpins essentially all Xbox 360 graphics work, this project included.
- **boma** — the original Xenia Canary patch set for Lost Odyssey, reimplemented here as runtime hooks.
- **re:Blue** — the Blue Dragon recompilation, which showed how a finished port on this SDK should look.
- **[LostOdysseyRecomp](https://github.com/freefrank/LostOdysseyRecomp)** by freefrank — another Lost Odyssey recompilation, whose published research located the game's Configuration screen task and System menu table used by the in-game settings and save-anywhere features. The implementations here are independent.
- **[SMAA](https://github.com/iryoku/smaa)** — by Jorge Jimenez, Jose I. Echevarria, Belen Masia, Fernando Navarro and Diego Gutierrez; used unmodified under its MIT licence.
- **[lzokay](https://github.com/jackoalan/lzokay)** — LZO decompression (MIT), used to read the game's menu textures.

---

## Legal

This repository contains **no game code, no game assets, and no executables** — only documentation and screenshots.

The in-game settings read the game's font and menu textures from the player's own copy at runtime; none of them are stored in this repository or in the port.

Lost Odyssey is © Microsoft / Mistwalker / Feelplus. This is an unaffiliated, non-commercial preservation and porting effort. Nothing here will ever distribute the game: any future release would require you to supply your own legally obtained copy.

Documentation in this repository is © its author. All rights reserved.
