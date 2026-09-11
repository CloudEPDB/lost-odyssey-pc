# Progress log

Newest first.

---

## September 2026

### All four discs, and discs as ISO or GOD — 11 Sep

Disc changes now work. When the game asks for another disc, the port finds it, mounts it and lets the game continue, with no prompt.

- The game requests discs from one place, through `XamSwapDisc`, which the SDK only stubs. The port wraps that call site, re-points the virtual filesystem at the requested disc and signals the event the game is waiting on. [Details →](technical.md#8-four-discs)
- Discs are recognised by their executable's header, so names do not matter. The intended layout is `data\disc1` … `data\disc4` next to the executable.
- Extracted folders, ISO images and Games on Demand packages are all read in place; nothing is extracted or copied, except the disc 1 executable when booting from an image.
- A missing disc produces a notice with a retry option instead of a hang.
- The in-game settings now read the game's font and textures through the same disc layer, so they work whichever form the discs are in.

Verified: a forced change from disc 2 to disc 1, and a full boot from an ISO image. Not yet verified: a change at a real chapter boundary, and Games on Demand packages.

Along the way, a folder labelled as disc 2 turned out to be a copy of disc 1. Reading the real image directly out of its zip showed that five archives differ between discs.

### Settings move into the game's own menu — 11 Sep

The game's Configuration screen now has four more tabs — Graphics, Patches, Extras, Textures — reached with **RB** from the native page.

- The screen is detected by wrapping its task in the game, which reports when it is interactive and when a native dialog covers it. The port only draws while the screen is interactive and uncovered.
- While a port tab is open, the game receives a controller at rest, and buttons still held on the way back to the native page are withheld until released.
- The tabs are drawn with the game's own font, menu textures and cursor, decoded at runtime from the player's data through a chain of index, archive, Unreal Engine 3 package, LZO-compressed tiled DXT5 textures and font glyph tables. [How →](technical.md#7-new-menu-pages-that-look-native)
- The layout was measured from a screenshot of the native screen, down to bevel colours, shadows and a help bar that squeezes its font horizontally.
- Every option from the F2 overlay is here, applied immediately where possible; changes that need a restart come with a confirmed restart.

### Save anywhere — 11 Sep

An optional toggle that enables Save in the System menu away from save points, using the game's own save flow. Tested by saving far from a save point, reloading in the same place and moving normally, and confirming that with the option off Save is greyed out again away from save points and still available at real ones.

Built by wrapping the menu's permission setter and its task rather than faking a save point. The table and functions involved were located by the published research of the [LostOdysseyRecomp](https://github.com/freefrank/LostOdysseyRecomp) project.

### SMAA 1x on both renderers — 11 Sep

The reference SMAA implementation, unmodified, runs as three compute passes on the final frame in Direct3D 12 and Vulkan, selectable next to FXAA. [Details →](technical.md#5-smaa-on-the-final-frame)

The Vulkan validation layers turned up a storage-image format mismatch that the driver had been hiding. Fixing it properly meant giving SMAA its own image in the presenter's output format. TAA was evaluated and postponed.

### Vulkan reaches parity — 7 Sep

Native 1080p and texture replacement now work on Vulkan as well as Direct3D 12. The two renderers are feature-equivalent.

Rather than copy the canvas pin — the most delicate code in the project, and the thing that makes 1080p work at all — it was extracted into one shared implementation both backends call. It turned out to touch no graphics API whatsoever: the entire backend-specific surface is a single lambda that marks the float constant buffers stale, two flags on D3D12 and two mask bits on Vulkan.

The same treatment for texture replacement: hashing, the pack index, PNG decoding and mip generation are now shared, and each renderer only supplies the part that is genuinely its own — creating the resource at the replacement's size and uploading the pixels.

Direct3D 12 was moved onto the shared code last, deliberately, so the risky step happened after it had been proven on Vulkan. Verified with no regression.

One ordering detail worth recording: the rules run in two phases, before and after the copy-mode check, because a resolve must not see the rewritten shader constants. The shared code preserves D3D12's original order rather than imposing a tidier one.

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

- A Linux build
- Retire the F2 overlay, now that the settings live in the game's own Configuration screen
- Verify disc changes at real chapter boundaries, and a full playthrough across the four discs
- Test Games on Demand packages
- Remaining UI polish: 1440p overlay artifacts, save list scrolling
- The two unported Xenia patches (ultrawide, debug menu)
