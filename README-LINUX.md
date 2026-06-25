# Building & running FFTPatcher / Shishi on Linux (Mono)

The FFTPatcher suite is a .NET Framework 3.5 WinForms project built for
Windows + Visual Studio. It runs natively on Linux under **Mono** — the
upstream code already guards Windows-only P/Invokes behind
`Utilities.IsRunningOnMono()`, so this is a supported path, not a hack.

This document covers what was needed to build and run it on a
case-sensitive Linux filesystem, with **Shishi Sprite Editor** as the
primary target. The other pure-managed tools (FFTPatcher, FFTorgASM,
EntryEdit, MassHexASM) build the same way.

## Prerequisites (Arch)

```bash
sudo pacman -S --needed mono mono-msbuild libgdiplus
```

- `mono` / `mono-msbuild` — runtime + MSBuild for .NET Framework targets.
  Modern `dotnet` **cannot** build these (Framework 3.5, Windows-only
  WinForms under modern .NET).
- `libgdiplus` — the GDI+ backend WinForms needs to render.

## The one source fix

**`ShishiSpriteEditor/Properties/Resources.resx`** referenced a handful of
embedded default-sprite files with lowercase paths
(`..\resources\battle\...`, `..\resources\sprite.ico`,
`..\resources\patcheddummyfolder.bin`) while the files on disk are
`Resources/BATTLE/...`, `Resources/sprite.ico`,
`Resources/PatchedDummyFolder.bin`.

Windows' case-insensitive filesystem hides this; a case-sensitive Linux
filesystem fails the build with `MSB3103: Invalid Resx file. Could not
find ...`. The fix is to correct those `ResXFileRef` values to match the
real on-disk casing. No symlinks, no renames — just the resx.

## Two build-environment workarounds (no source changes)

These are handled by the build commands below, not by editing the repo:

1. **Prebuild `zipResources.bat` is Windows-only.** `PatcherLib` and
   `PatcherLib.Resources` each embed a `Resources/Resources.tar.gz`
   (referenced from their `.resx`) that a prebuild `.bat` produces with
   `tar` + `gzip`. On Linux we generate those tarballs natively and skip
   the `.bat` by passing an empty `PreBuildEvent` property (Mono's targets
   fire the prebuild on `'$(PreBuildEvent)'!=''` and ignore
   `PreBuildEventUseInBuild`).

2. **Build only what Shishi needs.** Building
   `ShishiSpriteEditor.csproj` directly pulls in `PatcherLib` and
   `PatcherLib.Resources` and avoids the `ASMEncoding` / `MassHexASM`
   projects, whose old-style `$(MSBuildBinPath)\Microsoft.CSharp.Targets`
   import (wrong case + stale path) breaks under Mono's MSBuild. The full
   `.sln` also references the `FFTacText` project, which additionally needs
   the native `FFTTextCompression.dll` (not built on Linux).

## Build

```bash
cd vendor/FFTPatcher

# 1. Generate the two embedded resource tarballs (replaces zipResources.bat)
( cd PatcherLib/Resources && rm -f Resources.tar* && \
  tar -cf Resources.tar --exclude '*[._]svn' * && gzip -9 Resources.tar )
( cd PatcherLib.Resources/Resources && rm -f Resources.tar* && \
  tar -cf Resources.tar --exclude '*[._]svn' --exclude '*.xls' * && gzip -9 Resources.tar )

# 2. Build Shishi (+ PatcherLib, PatcherLib.Resources). Empty PreBuildEvent
#    skips the Windows-only prebuild .bat.
msbuild ShishiSpriteEditor/ShishiSpriteEditor.csproj \
  /p:Configuration=Release /p:Platform=x86 "/p:PreBuildEvent=" /m
```

Output: `ShishiSpriteEditor/bin/x86/Release/ShishiSpriteEditor.exe`.

The generated `Resources.tar.gz` tarballs and `bin/` output are gitignored
build artifacts — they are not committed.

## Run

```bash
mono vendor/FFTPatcher/ShishiSpriteEditor/bin/x86/Release/ShishiSpriteEditor.exe
```

A WinForms window opens (X11 / Xwayland). Verified working on Mono 6.12 +
libgdiplus.

## Not yet ported to Linux

- **FFTacText** — needs the native `TextCompression/TextCompression.vcproj`
  (`FFTTextCompression.dll`), a Windows C++ build.
- **ASMEncoding / MassHexASM** — `Microsoft.CSharp.Targets` import needs the
  same lowercase/`$(MSBuildToolsPath)` correction the other projects already
  use before they'll build under Mono.
