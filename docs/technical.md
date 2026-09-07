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

*More to come as the port progresses.*
