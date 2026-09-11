# Technical notes

Notes on the parts of this port that were genuinely hard. Written for anyone doing the same thing to another Xbox 360 title — most of what follows is not specific to Lost Odyssey.

---

## 1. Why 1080p is hard on a 360 game, and what it takes

### The constraint

The Xbox 360's GPU does not render into main memory. It renders into 10 MB of embedded DRAM (EDRAM) attached to the GPU daughter die, and only resolves finished tiles out to RAM. That 10 MB is the hard budget for colour plus depth, simultaneously.

EDRAM is addressed in tiles of 80×16 samples at 32 bits per pixel — 5120 bytes each. 10 MB is exactly **2048 tiles**.

At 1280×720:

```
16 tiles across × 45 down = 720 tiles
colour + depth            = 1440 tiles   ✓ fits
```

At 1920×1080:

```
24 tiles across × 68 down = 1632 tiles
colour + depth            = 3264 tiles   ✗ needs 16 MB
```

This is *why* Lost Odyssey renders at 720p. It is not an engine limitation or an artistic choice; 1080p does not physically fit in the console. Every 360 game that renders at 1080p either uses a single buffer format that fits, or renders in predicated tiles.

Under emulation, the EDRAM is just a buffer in host memory — so you can make it bigger. The problem is everything downstream that assumed it wasn't.

### Step 1 — enlarge the EDRAM, and the fields that address it

The fork raises the emulated EDRAM to **16384 tiles (80 MB)** and widens the render-target base fields in `RB_COLOR_INFO` / `RB_DEPTH_INFO` from the console's 11 bits to 13. That alone gets you a buffer large enough. It also breaks the resolve path.

### Step 2 — the resolve shaders don't know about your bigger buffer

The GPU plugin ships **precompiled** compute shaders that copy resolved tiles out of EDRAM. They take the source tile base as a shader constant, and that constant is an **11-bit field** — because on real hardware it can never exceed 2048. Any base past 2048 wraps around and you resolve from the wrong place.

Recompiling those shaders was not an option (they're shipped as compiled blobs). Two things made it tractable:

**A sliding window.** Instead of pointing the shaders at an 80 MB buffer, the resolve and clear paths bind a **2048-tile window** into it, and pass a base relative to that window. The window is chosen per resolve from the tile the operation actually touches. The shaders keep their 11-bit world view and never know the buffer is forty times larger.

On D3D12 this is a matter of descriptor offsets. On Vulkan it needs **eight descriptor sets**, one per window, each with its own `VkDescriptorBufferInfo::offset` — and a subtlety that cost a debugging session: the Vulkan clear path issues *two* dispatches, depth and colour, with the descriptor bound once before both. Depth and colour can live in different windows. It has to be rebound per dispatch. (D3D12 uses two separate descriptors, so the bug doesn't exist there.)

**Binary-patching the modulus.** The shaders also carry a hardcoded wrap constant — `2048 tiles × 1280 dwords = 2621440` — used to fold addresses back into EDRAM. That single literal has to become the real buffer size.

For **DXBC** this means finding the constant, replacing it, and recomputing the container's MD5-derived checksum, or the driver rejects the blob. For **SPIR-V** it is much easier: SPIR-V has no checksum, so it's a matter of walking the instruction stream, locating the one `OpConstant` with that literal, and rewriting the word. Both patchers assert that exactly one candidate exists, so a future SDK version that changes the shaders fails loudly instead of silently corrupting them. The results are validated with `spirv-val`.

The scaled shader variants are deliberately left alone: at supersampled presets the guest still renders its original 720p canvas, never exceeds 2048 tiles, and wrapping there is correct behaviour.

### Step 3 — the canvas pin, or: why your HUD is in the corner

This is the part that surprises people, and it is worth understanding before attempting a native-resolution patch on any 360 game.

Enlarge the render target and the 3D scene fills it correctly — the game's own projection matrices are resolution-independent. But the **2D layer does not move**. Menus, HUD, subtitles and full-screen effects are authored against a fixed 1280×720 canvas, with orthographic projections whose constants bake those dimensions in. Point that at a 1920×1080 target and the interface renders at its original pixel size, anchored in one corner, surrounded by empty space.

You cannot fix this by scaling the output, because the 3D and 2D layers need *different* treatment in the same frame. It has to be done per draw call, which means: intercept each draw, work out whether it belongs to the 2D layer, and if so rewrite its state to target the larger canvas.

The fork does this with a set of rules applied inside the draw path, roughly 545 lines across three hook points in the command processor — three, rather than one, because the rules need different context: one of them requires the vertex shader to have been analysed already, since it decides by inspecting the projection's span.

The rules cover: detecting orthographic 2D draws from their projection constants, un-shrinking viewports that were sized for the small canvas, scaling scissor rectangles, correcting NDC mapping, and invalidating the shader constant buffers afterwards so the rewritten values actually reach the GPU.

Notably, this code touches **no graphics API at all** — it manipulates the emulated GPU register file. Porting it from D3D12 to Vulkan is a single-line difference: where D3D12 marks its float constant buffers stale with a flag, Vulkan clears bits in a bitmask. That is the entire backend coupling of the most intricate code in the project.

---

## 2. A build trap worth knowing about

Cost: four failed launches and a crash that appeared in code that had not been touched.

The GPU plugin is a fork that overrides SDK headers by placing its own copies **earlier on the include path**. Adding a *new* override header creates a trap:

The already-compiled object files have dependency files (`.d`) that reference the *SDK's* header — the one that hasn't changed. Ninja checks those, sees nothing newer, and recompiles only the `.cpp` you edited.

If the new header changes the size or member layout of a class, you now have some objects using the old layout and some using the new one. In this case one translation unit allocated a render target cache using the old `sizeof`, while its constructor wrote eight descriptor sets past the end of it. The result was heap corruption, a crash in untouched code, and — the part that makes it so hard to diagnose — no Vulkan validation layer error at all, because nothing invalid was ever submitted to the API.

**The rule:** after adding a shadow header to a fork, delete the build directory and rebuild everything. Editing an existing one propagates correctly; adding a new one does not.

**The symptom to recognise:** a crash that lands between two log points separated only by code you did not modify.

---

## 3. Choosing a Vulkan render target path

The Vulkan backend inherits two strategies for emulating EDRAM:

- **Host render targets** — real framebuffers, with copies between them to emulate EDRAM aliasing.
- **Fragment shader interlock (FSI)** — exact emulation inside a storage buffer.

The default configuration falls back to host render targets. In Lost Odyssey this is the wrong choice twice over: the game's dynamic shadows round-trip through EDRAM to a texture and back, which the copy-based path renders incorrectly, and the copies cost enough performance to put a supersampled preset under 30 fps.

Forcing the FSI path fixes both. This was initially misdiagnosed as "Vulkan is just poorly optimised" — worth flagging, because the symptom (broken shadows *and* bad framerate) looks like two separate problems and is one setting.

---

## 4. Patches as hooks, not bytes

The recompilation SDK has no facility for patching bytes at runtime — the guest code is gone by then, compiled into the host binary. What it has instead are mid-instruction hooks: run host code at a given guest address, with access to the guest registers, optionally skipping or redirecting the original instruction.

This turns out to be *better* than byte patching for distributing game fixes. Each of the Xenia Canary patches becomes a hook guarded by a boolean, which means every one of them is a toggle in the options menu that takes effect on the next frame, rather than a permanent modification chosen before launch.

The catch is verification: the hook has to sit at an address that means the same thing in the recompiled output as it did in the original executable. Addresses must be checked against the generated assembly, and against the correct regional build — the first patch list tried here was for the Japanese release and pointed into unrelated code.

---

## 5. SMAA on the final frame

SMAA 1x runs where the emulator's FXAA already did: on the frame about to be presented, after the guest has finished drawing. It is the reference implementation, unmodified, compiled into three compute passes — edge detection, blending weights, neighbourhood blending — with its area and search lookup textures uploaded once. A single HLSL source produces DXBC for Direct3D 12 and SPIR-V for Vulkan.

Running pixel-shader code as compute needed two adjustments. The edge detection pass uses `discard`, which compute does not have; it is redefined to write zero weights into a target that starts cleared, which is exactly what discarding would have left there. And every texture read becomes an explicit level-0 sample, because compute has no derivatives.

The Vulkan validation layers caught a real bug that the NVIDIA driver was quietly tolerating. The presenter's gamma pass declares its storage image as `rgb10_a2`, and the first version pointed it at the FXAA source image, which is RGBA16F — undefined behaviour that happened to look correct. SMAA now has its own intermediate image in the presenter's guest output format, and the validation output is clean.

Applying SMAA to the final frame means the HUD is antialiased too. For an interface made of flat 2D art that is harmless, and it keeps the effect independent of the game's own render passes.

TAA was evaluated and not attempted yet. Doing it properly on a 360 game means injecting camera jitter into specific shaders, capturing the scene before the HUD is drawn, and having depth, history and motion vectors — a different order of work from a post-process pass.

---

## 6. Wrapping whole guest functions

Section 4 described mid-instruction hooks. Some features need to act *around* a function instead — before it runs, after it returns — and the SDK allows that too, although it is not presented as a feature: every recompiled function is emitted as a weak alias of its implementation. Defining a function with the same name in the project replaces it at link time, and the original stays callable under its implementation name.

```cpp
REX_EXTERN(__imp__sub_82B88020);  // the recompiled original

REX_EXTERN(sub_82B88020) {         // replaces it at link time
  // ...before...
  __imp__sub_82B88020(ctx, base);
  // ...after...
}
```

No generated code is edited, nothing is byte-patched, and it links without duplicate symbols. Three features in this port are built this way:

- **Save anywhere** wraps the System menu's permission setter and its menu task, so the Save row stays enabled away from save points and the game's own permission comes back when the option is turned off.
- **The settings tabs** wrap the Configuration screen's task, to know every frame whether the screen is open and interactive.
- **Disc changes** wrap the game's one call site of `XamSwapDisc`.

---

## 7. New menu pages that look native

The settings tabs are drawn by the port, on top of the game's own Configuration screen, and are meant to be indistinguishable from it. That breaks down into three problems: knowing when to draw, taking the controller, and looking right.

**When.** The Configuration screen is a task object in the game. One of its fields reaches an "interactive" state once the opening animation has finished, and another is non-zero while a native dialog is open on top of it. Wrapping the task (section 6) reads both every frame. The tabs are only shown while the screen is interactive with nothing above it, and vanish the moment that stops being true, so they never fight the game's own transitions or dialogs.

**The controller.** The game reads the pad in two places, and both pass through one hook. While one of the port's tabs is open, the hook reads the buttons for the page and hands the game a pad at rest, so the native page underneath does not react. Buttons still held when switching back to the native page are withheld until they are released — otherwise the same B press that returns to the game's options would also close them.

**Looking right.** An imitation with a similar font was never going to pass, so the port uses the real assets, read from the player's own game data when it starts: the menu font, the title font, and the texture atlas that holds the brushed-metal panels, the curved corner of the side panel, and the cursor. Reaching them means walking a chain of formats:

1. the disc's file index, whose names are packed in base 40 against a shared dictionary;
2. an archive whose entries use a custom, bit-oriented LZ compression;
3. an Unreal Engine 3 package, big-endian;
4. inside it, `Texture2D` objects holding DXT5 data that is LZO-compressed, stored in the Xbox 360's tiled block order and byte-swapped per 16-bit word — and `Font` objects, glyph tables laid over those textures.

Every step was validated first with a throwaway Python prototype that decoded the assets to PNG files, before any of it went into the executable.

The layout was then measured from a screenshot of the native screen, pixel profile by pixel profile: panel edges and bevel colours, the inset of the selected value, the drop shadow under the highlighted row, the exact scale of each font, and the fact that the help bar uses the same font squeezed horizontally. Highlighted and inactive text are drawn with recoloured copies of the font pages: the game's glyphs are a white face with a black outline, and no multiplicative tint can turn that into the dark face with a light outline that the game uses on a highlighted row.

None of the game's assets are in this repository or in the port. If the data cannot be read, the page falls back to a plain style instead of failing.

---

## 8. Four discs

Each of Lost Odyssey's four discs carries its own copy of the executable — the same code, with a header that says "disc N of 4" — and its own set of archives. Comparing the discs file by file, five archives differ from one disc to the next: the file index, events, field data, video and sound. The rest are identical.

When the game needs another disc, it calls `XamSwapDisc` with the disc number and an event to signal once the disc is in, waits on that event, and then re-reads the file index to check it has the disc it asked for. In the SDK, `XamSwapDisc` is a stub that reports success and never signals anything.

The port wraps the game's only call site. After the original runs, it looks the requested disc up, re-points the `game:` and `d:` links of the virtual filesystem to a device over that disc, and signals the event. The game's own check then passes, and it carries on. Devices for previous discs stay registered, since the game may still hold files open on them.

Discs are identified by that executable header, never by name. The same catalogue accepts every common form, each read in place by one of the SDK's filesystem devices: an extracted folder, an XDVDFS ISO image, or a Games on Demand package. Booting from an image has one wrinkle: the SDK needs the entry executable to be a file inside a folder, so only `default.xex` (6 MB) is copied to the cache, and the image is mounted over it as soon as the runtime exists.

Two lessons from getting there:

- **Mount order matters.** Mounting a different disc early in startup crashed the game on launch. The SDK still reads `game:\default.xex` while it prepares the module, and at that moment it found another disc's executable. Anything that changes the mounted disc has to wait until the module is prepared.
- **Check what you are given.** A folder labelled `disc2` on the development machine turned out to be a second copy of disc 1: every file hashed identical. Reading the real disc 2 image straight out of its zip archive — streaming, without extracting it — is what showed which files genuinely differ, and it is also why the port trusts executable headers rather than names.

Validated so far: a disc change forced by booting with disc 2 mounted, where the game immediately asked for disc 1, got it, and continued; and a full boot from an ISO image, including the in-game settings reading their assets from it. Not yet exercised: a disc change at a real chapter boundary, a change between ISO images, and Games on Demand packages.

---

*More to come as the port progresses.*
