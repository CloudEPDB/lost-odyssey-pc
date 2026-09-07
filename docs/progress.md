# Progress log

Newest first.

---

## September 2026

### Vulkan renderer — 6 Sep

Both backends now live in a single GPU plugin and are selectable from the options menu without reinstalling anything.

- Rebuilt the recompilation SDK from source with Vulkan enabled. The distributed Windows package ships with it compiled out, so this was unavoidable — the Vulkan presenter lives in the runtime, not in the plugin.
- Vulkan and Direct3D 12 now coexist in one 6.5 MB DLL. The runtime would normally pick D3D12 unconditionally; the app loads the plugin itself so the menu can decide.
- Fixed broken dynamic shadows and a framerate collapse on Vulkan. Both were one setting: the render target path was defaulting to host framebuffers instead of exact EDRAM emulation. [Details →](technical.md#3-choosing-a-vulkan-render-target-path)
- Ported the EDRAM sliding window to Vulkan — eight windowed descriptor sets, with the clear path rebinding per dispatch.
- Wrote a SPIR-V patcher for the resolve shaders, mirroring the existing DXBC one. Output validated with `spirv-val`.

Still outstanding on Vulkan: 1080p native (needs the canvas pin ported) and texture replacement.

Also investigated and ruled out [XenosRecomp](https://github.com/hedge-dev/XenosRecomp) as an alternative shader strategy — it translates shaders ahead of time, which does not fit a runtime-translating plugin.

**Why Vulkan matters here:** it is the prerequisite for Linux. The SDK supports Linux including arm64; Android it does not support at all.

### Texture replacement, and DualSense glyphs — 5 Sep

Texture dumping to PNG, a replacement pack loaded from disk, and hot reload on **F7** without restarting the game. Textures are keyed by a content hash (XXH3) rather than by address, so packs survive across sessions and saves.

The first thing that went through it: the button glyph atlas. On-screen prompts can now show DualSense glyphs instead of the Xbox buttons the original release hardcoded.

### Options menu and turbo — 3–4 Sep

- An in-game options menu on **F2**: resolution presets, antialiasing, patches, turbo, textures. Writes the config file itself and relaunches the process when a setting requires it.
- Turbo at 1.5×/2×/3×, hold or toggle, bindable to a controller button.
- Proper application icon.

---

## August 2026

### 1080p native on Direct3D 12 — late Aug

The headline feature, and the hardest. The game now renders a real 1920×1080 frame rather than a stretched 720p one.

- Emulated EDRAM enlarged from 2048 to 16384 tiles, with widened render-target base fields.
- A 2048-tile sliding window over that buffer, so the precompiled resolve shaders keep working.
- The resolve shaders' hardcoded wrap constant binary-patched, with the DXBC checksum recomputed.
- The canvas pin: rules applied per draw call that detect the game's 2D layer and retarget it at the larger canvas, so the HUD and menus follow the resolution instead of sitting in a corner at original size.

[Full write-up →](technical.md#1-why-1080p-is-hard-on-a-360-game-and-what-it-takes)

### Game patches as runtime toggles — 30 Aug

The Xenia Canary patch set (original work by **boma**) reimplemented as recompiler hooks instead of byte patches, which makes every one of them a switch that can be flipped while playing: 60 fps, character flicker fix, occlusion queries, post-process upscale fix, depth of field, motion blur, 16× anisotropic, dynamic shadows.

Two patches from that set remain unported — 21:9 / 16:10 support and the debug menu — because they need data writes at startup rather than instruction hooks.

*Note for anyone following along: patch addresses are region-specific. The first list tried here was for the Japanese build and pointed at unrelated code.*

### Playable — 29–30 Aug

First fully playable build: main menu, gameplay, saves, achievements.

- Fixed a crash that killed the game after roughly 27 minutes (heap allocation failure).
- Supersampling working via the plugin's render scale.
- Established the workflow for unresolved guest functions: the log names the address, it goes in the manifest, recompile.

---

## Planned

- Canvas pin on Vulkan → 1080p native on both renderers
- Texture replacement on Vulkan
- A Linux build
- Move the F2 settings into the game's own configuration screen, and retire the overlay
- Remaining UI polish: 1440p overlay artifacts, save list scrolling
- The two unported Xenia patches (ultrawide, debug menu)
