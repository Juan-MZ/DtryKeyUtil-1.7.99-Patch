# DtryKeyUtil — 1.7.99 Compatibility Patch

**This is not a mod. It is only the recompiled DLL.**
**It REQUIRES the original mod installed** — [DtryKeyUtil](https://www.nexusmods.com/skyrimspecialedition/mods/69944) by Dtry (D7ry).

**This patch's source:** <https://github.com/Juan-MZ/DtryKeyUtil-1.7.99-Patch>
**Upstream (D7ry's original):** <https://github.com/D7ry/DtryKeyUtil> (last commit 2022-11-01)

Not affiliated with D7ry or Bethesda.

## Permissions — no request was needed

The mod page's **Modification permission** is explicit:

> *"You are allowed to modify my files and release bug fixes or improve on the features
> **without permission from or credit to me**."*

That grant is what this patch relies on, since the upstream repository carries no licence file.
The **Upload permission** setting forbids uploading the file to *other sites* — Nexus is not
"another site", and this patch is not mirrored anywhere else, so that is respected.

Credit is given to D7ry throughout anyway, even though the permissions do not require it.

## Who needs this

You are on **Skyrim SE/AE 1.7.99** (the 20 August 2026 update) and DtryKeyUtil throws:

```
failed to open address library file
```

If you are still on 1.6.x or 1.5.97, you do **not** need this patch — keep using the official
release.

## Requirements

- **DtryKeyUtil** (the original mod — required; it also provides the
  `SKSE/Plugins/dtryKeyUtil/config/` settings tree)
- SKSE64 **2.3.0** (the 1.7.99 build)
- Address Library for SKSE Plugins (updated — must include `versionlib-1-7-99-0.bin`)
- Any mod that depends on DtryKeyUtil, since this is a utility plugin

## Installation

Use **`DtryKeyUtil-1.7.99-Compatibility-Patch.zip`** as-is, without extracting it.

### Vortex

1. Install the original mod normally first.
2. Mods tab → *Install From File* → pick the zip → Enable.
3. Vortex will report a **file conflict** with the original mod (both ship the same DLL).
   When prompted, set this patch to *Load After* the original mod so **this patch wins**.

> If you pick the other option, Vortex keeps deploying the old DLL and it will look like the patch
> did nothing.

### Mod Organizer 2

*Install a new mod from an archive*, then place this patch **below** the original mod in the left
pane (in MO2, lower wins).

### Manual

Copy `SKSE\Plugins\DtryKeyUtil.dll` over the existing one — only if you installed the mod manually.
With Vortex or MO2, always go through the manager.

### Verifying it worked

Launch the game. If it reaches the main menu with no error box, it worked. The plugin writes
`Documents\My Games\Skyrim Special Edition\SKSE\DtryKeyUtil.log`, which should log
`DtryKeyUtil ... has finished loading.` with no address library errors.

## What was broken

Skyrim 1.7.99 shipped a new Address Library format (**format 5** — a 96-byte header and a dense
array indexed directly by ID; a 1.7.99 `versionlib-1-7-99-0.bin` starts with the bytes
`05 00 00 00`). The released DLL was built against a much older CommonLibSSE-NG that only
understands formats 1 and 2, so it cannot read the file at all.

1.7.99 also recompiled Skyrim and moved several internal structures, which needs an updated
CommonLib to model correctly.

## What was changed

- **Rebuilt against CommonLibSSE-NG 6.7.0** — format 5 support plus the 1.7.99 layouts. The
  `commonlibsse-ng-flatrim` vcpkg port the mod used is far too old, so CommonLib is now built from
  source. ("flatrim" meant SE+AE without VR, so VR stays off here to match.)
- **`src/utils.h`** — `logger::info()` was being handed a runtime string as its *format* string.
  Newer fmt rejects that at compile time, and it was a latent bug regardless: an INI value
  containing braces would have been parsed as a format specifier.
- **Plugin declaration written by hand.** CommonLib's `add_commonlibsse_plugin()` helper generates a
  `SKSEPluginInfo()` declaration whose `StructCompatibility` field occupies the same offset as
  `versionIndependenceEx`, so it can only ever write 0 or 1 — it cannot set the Address Library v5
  compatibility flag. The declaration now uses `SKSE::PluginVersionData` directly.
  `PluginDeclaration::GetSingleton()`, which the mod calls for its name and version, still works:
  it just reads the `SKSEPlugin_Version` export, and the two layouts agree on those fields.
- **`vcpkg.json`** — updated baseline, dropped the unused `boost-stl-interfaces` and
  `rsm-binary-io` dependencies and the `fmt` 8.0.1 pin.

**No feature was added, removed, or redesigned.** The DLL still reports itself as version 0.0.2 and
exports the same three symbols as before (`SKSEPlugin_Load`, `SKSEPlugin_Query`,
`SKSEPlugin_Version`).

## Verification

Checked against a real 1.7.99 `SkyrimSE.exe` and its `versionlib-1-7-99-0.bin`:

- The mod uses **no** hardcoded offsets — no `RELOCATION_ID`, no `write_call`/`write_branch`, no
  `VariantOffset`. Its only hook is a vtable replacement.
- That hook targets `RE::VTABLE_MovementHandler` (AE id 208715), which **resolves on 1.7.99** to
  RVA `0x19351A8`, and its first entries all point into `.text` — a valid vtable.
- The hooked slots `0x2` and `0x4` match `ProcessThumbstick` and `ProcessButton` in CommonLibSSE-NG
  6.7.0's `PlayerInputHandler` virtual layout, and both slots hold valid code pointers on 1.7.99.
- The built DLL declares `versionIndependenceEx = 3` (struct-independent + Address Library v5) and
  `versionIndependence = 1` (uses Address Library), read back out of the binary.

**It has not been play-tested in-game.** Back up your save before using this on a main playthrough,
and please report anything odd — especially movement input handling, since that is what the vtable
hook touches.

## Building from source

`source/DtryKeyUtil-1.7.99-compat.patch` is the diff against upstream.

```sh
git clone https://github.com/D7ry/DtryKeyUtil.git
cd DtryKeyUtil
git apply ../DtryKeyUtil-1.7.99-compat.patch
git clone --depth 1 --branch ng https://github.com/alandtse/CommonLibVR.git external/CommonLibNG
set VCPKG_ROOT=<path to vcpkg>
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=RelWithDebInfo ^
  -DCMAKE_TOOLCHAIN_FILE=%VCPKG_ROOT%\scripts\buildsystems\vcpkg.cmake ^
  -DVCPKG_TARGET_TRIPLET=x64-windows-static-md -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreadedDLL
cmake --build build --config RelWithDebInfo
```

Requires Visual Studio 2022 (or Build Tools) with MSVC v143, Windows SDK 10, CMake and vcpkg.
Built with CommonLibSSE-NG 6.7.0 (commit `3d81614`), MSVC 14.44, Windows SDK 10.0.26100.

`symbols/DtryKeyUtil.pdb` (full download only) makes crash logs show function names. It is not
needed to play.

## Credits

- **D7ry (Dtry)** — creator of DtryKeyUtil. The mod is entirely their work.
- **Brodie Thiesfield** — SimpleIni (MIT), bundled in the mod's source.
- **meh321** — Address Library for SKSE Plugins.
- **CommonLibSSE-NG** contributors (CharmedBaryon, alandtse, powerof3, Ryan-rsm-McKenzie) — the
  1.7.99 and format-5 support this patch depends on.
- **The SKSE team** — SKSE64.

Made for the community. Nothing asked in return. If D7ry updates the mod, use their version — this
patch only exists so the mod can keep being used in the meantime.
