Diablo IV on Linux: fixing the FindNextFileNameW / kernel32 error 127 crash
                                 
  Symptom: After Diablo IV's July 1, 2026 update, the game/Battle.net fails to launch on Linux (Steam Deck, desktop Linux, etc.) with:

  Failed to load function.
  Module: KERNEL32.dll
  Import: FindNextFileNameW
  Error code: 127
  Procedure not found.

  Root cause: That update started calling FindNextFileNameW, a Windows kernel32.dll API that Wine/Proton has never implemented. Since no current Wine build exports it, the game's import table can't be resolved and it refuses to start. Confirmed to affect all Proton/Wine builds as of the patch date — this isn't a
  config problem, the function genuinely doesn't exist yet.

  Things that did not fix it: switching Proton versions (Proton Experimental bleeding-edge, GE-Proton10-34, UMU-Proton) — none of the currently released builds include the missing export yet.

  The fix

  A community patch (LinkeTh/proton-cachyos-diablo4-findnextfilenamew) adds a minimal stub: it exports FindNextFileNameW from kernelbase.dll and forwards kernel32.dll's export to it. The stub just reports "no more hard-link names" (FALSE / ERROR_HANDLE_EOF) — good enough to satisfy Diablo IV's loader, not a full
  implementation. Treat it as temporary; drop it once Wine/GE-Proton/Blizzard ship a real fix.

  The repo's README assumes CachyOS Proton is already installed as a base to patch. If you don't have that (most people don't), you can graft the same two DLLs onto any Wine-based Proton build you already have (GE-Proton, in this case) instead.

  Steps

  1. Install build tools (autoconf, mingw-w64 for the PE/Windows cross-compiler). On an immutable/atomic distro (Bazzite, SteamOS-like), use Homebrew/linuxbrew rather than touching the base image:
  brew install autoconf mingw-w64
  1. On a traditional distro, use your package manager (e.g. sudo pacman -S base-devel per the repo's own instructions, or the Fedora/Debian equivalents plus a mingw-w64 package).
  2. Clone the patch repo and pull in just the Wine submodule:
  git clone --branch diablo4-findnextfilenamew-workaround \
    https://github.com/LinkeTh/proton-cachyos-diablo4-findnextfilenamew.git
  cd proton-cachyos-diablo4-findnextfilenamew
  git submodule update --init --depth 1 wine
  git submodule update --init --depth 1 Vulkan-Headers   # needed to regenerate wine/vulkan.h
  git -C wine apply ../patches/wine/0001-kernel32-export-FindNextFileNameW.patch
  3. Generate Wine's build scripts and headers (the checked-out submodule doesn't ship a pre-built configure):
  cd wine
  cp ../Vulkan-Headers/registry/vk.xml ../Vulkan-Headers/registry/video.xml .
  ./autogen.sh
  cd ..
  4. Configure a minimal 64-bit-only build (Diablo IV is 64-bit, so we skip almost everything else — no X11, Vulkan, audio, etc., since we only need two DLLs to compile):
  mkdir ../wine-diablo-build64 && cd ../wine-diablo-build64
  ../proton-cachyos-diablo4-findnextfilenamew/wine/configure \
    --enable-win64 --disable-tests \
    --without-alsa --without-cups --without-dbus --without-fontconfig \
    --without-freetype --without-gettext --without-gnutls --without-gstreamer \
    --without-krb5 --without-openal --without-opencl --without-opengl \
    --without-oss --without-pcap --without-pulse --without-sane --without-sdl \
    --without-udev --without-unwind --without-usb --without-v4l2 \
    --without-vulkan --without-wayland --without-x
  5. Build just the two target DLLs (note: the top-level make targets need the full output path, not just the module name):
  make -j"$(nproc)" \
    dlls/kernelbase/x86_64-windows/kernelbase.dll \
    dlls/kernel32/x86_64-windows/kernel32.dll
  6. Graft the patched DLLs onto a copy of an existing Proton/Wine build (don't overwrite your working install — make a new Steam compatibility tool):
  SOURCE="$HOME/.local/share/Steam/compatibilitytools.d/GE-Proton10-34"
  DEST="$HOME/.local/share/Steam/compatibilitytools.d/GE-Proton10-34-diablo-findnext"
  cp -a "$SOURCE" "$DEST"

  # Only replace the 64-bit copies — leave syswow64 (32-bit) alone, since we only built 64-bit DLLs
  for f in kernel32.dll kernelbase.dll; do
    install -m 0644 "../wine-diablo-build64/dlls/${f%.dll}/x86_64-windows/$f" \
      "$DEST/files/lib/wine/x86_64-windows/$f"
    install -m 0644 "../wine-diablo-build64/dlls/${f%.dll}/x86_64-windows/$f" \
      "$DEST/files/share/default_pfx/drive_c/windows/system32/$f"
  done

  # Give it a distinct name so it's easy to pick in Steam/Lutris
  sed -i 's/"GE-Proton10-34" \/\/ Internal name of this tool/"GE-Proton10-34-diablo-findnext" \/\/ Internal name of this tool/' "$DEST/compatibilitytool.vdf"
  sed -i 's/"display_name" "GE-Proton10-34"/"display_name" "GE-Proton10-34: Diablo IV FindNextFileNameW fix"/' "$DEST/compatibilitytool.vdf"
  7. Fully quit and restart Steam (or your launcher — this also worked from Lutris/umu-run without any Steam involvement, since it just scans the same compatibilitytools.d folder), then force Diablo IV / Battle.net to use the new tool:
    - Steam: Diablo IV → Properties → Compatibility → Force the use of a specific Steam Play compatibility tool → select the new one.
    - Lutris: set the Wine/Proton version for the Battle.net install to the new tool.

  Result

  Confirmed working: Battle.net launches and logs in successfully using this patched build.

  Caveat: this is a stopgap, not a proper fix. Watch for an official GE-Proton/Wine release or a Blizzard patch that resolves this natively, then switch back to your normal Proton build and delete the custom tool directory.
