# FAQ

### Where is the source code?

Not published yet.

The port is in active development and large parts of it are still moving — the renderer fork in particular. Publishing it in this state would mean half-working forks and broken builds circulating under the project's name while the real thing is still being fixed. The source will be opened when the port is finished enough that what people build actually represents it.

In the meantime this repository exists so the work is visible: what it does, how it does it, and what is left.

### Will there be a download?

Eventually, of a *patcher* — never of the game.

Any release would require you to supply your own legally obtained copy of Lost Odyssey. No game code, no game assets, and no recompiled executable will ever be distributed here. That is not only a legal position, it's the practical one: it is what keeps projects like this online.

### Why not just use Xenia?

Xenia is an excellent emulator and this project would not exist without the work behind it. They solve different problems.

An emulator translates the game's code while it runs. A static recompilation translates it *once*, ahead of time, into a normal PC executable. The result runs the game's logic as native x86-64 with no interpreter or JIT in the way — but more importantly for this project, it makes the game *modifiable* in ways an emulator cannot easily match:

- Fixes become toggles in an options menu instead of external patch files.
- The renderer is part of the build, so it can be changed — which is how a game hard-limited to 720p by console memory ends up rendering native 1080p.
- The output is a single executable that behaves like a PC game.

The cost is that the work is per-title and substantial. Xenia runs thousands of games; this runs one.

### Does it run better than the console?

Yes, in the ways you would expect from native code and a modern GPU: 60 fps, higher resolutions, anisotropic filtering, supersampling, and no loading from optical media.

### Linux? Steam Deck? Android?

Linux is planned and is the reason the Vulkan renderer exists — the recompilation SDK supports Linux, including arm64. It has not been built yet.

Android is not supported by the SDK, so it is not on the table.

### Can I help / can I test it?

Not yet, but this is worth asking again later. Watch the repository for updates.

### Is this affiliated with Microsoft, Mistwalker or Feelplus?

No. This is an unaffiliated, non-commercial preservation effort. Lost Odyssey is © its respective rights holders.
