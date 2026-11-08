# Nebula Music Player — FFmpeg Build & Relinking Guide

This document describes how the FFmpeg-based AC-3 / E-AC-3 decoder shipped with Nebula Music Player was built and how an independent developer can rebuild and relink the FFmpeg JNI library. It applies to the **`ffmpeg-eac3-v1.0`** release and does **not** include the private source code of Nebula Music Player.

## Source, release, and licenses

- **Public release:** https://github.com/Nebula-dev-15/nebula-ffmpeg-licenses/releases/tag/ffmpeg-eac3-v1.0
- **FFmpeg Git commit:** `573f2b81ebd97da81def2ba762e81b0328547004` (FFmpeg release/6.0 source)
- **AndroidX Media3 Git commit:** `8c6678b657ede1e7883fc164ef73ed483c7796c3` (Media3 1.11.1)
- **Media3 FFmpeg module:** `libraries/decoder_ffmpeg` (Media3 source licensed separately under Apache 2.0)
- **FFmpeg license:** LGPL v2.1 or later; see `COPYING.LGPLv2.1` and `LICENSE.md` in the corresponding FFmpeg source archive.
- **Published assets:**
  - `ffmpeg-source-573f2b81.tar.gz` — FFmpeg corresponding source and license files.
  - `ffmpeg-relink-materials.tar.gz` — FFmpeg static archives (`.a`) and the Media3 FFmpeg JNI wrapper object file (`ffmpeg_jni.cc.o`) for each target ABI.
- **Decoder capabilities:** AC-3 (`ac3`) and E-AC-3 (`eac3`) audio decoders enabled.
- **Target Android API:** 23; Android ABIs: `arm64-v8a`, `armeabi-v7a`, `x86`, `x86_64`.

## Build environment used

The successful build was made in Linux (Ubuntu on WSL2), with:

- Linux Android SDK command-line tools and SDK packages, **NDK r28c (`28.2.13676358`)**, Java 17.
- Git, Bash, CMake, Ninja, curl, unzip/zip, Python 3, and C/C++ build tools.
- An Internet connection for obtaining pinned Media3 sources and Gradle dependencies.

The Android SDK/NDK must be **Linux-hosted** when using WSL2. Do not point the NDK variable to a Windows-hosted NDK installation. The steps below assume the SDK is installed in `$HOME/android-sdk` and NDK r28c is present there; change these paths for your machine.

```bash
sudo apt update
sudo apt install -y git cmake ninja-build unzip zip openjdk-17-jdk python3 build-essential curl

export ANDROID_HOME="$HOME/android-sdk"
export ANDROID_SDK_ROOT="$ANDROID_HOME"
export NDK_PATH="$ANDROID_HOME/ndk/28.2.13676358"

"$NDK_PATH/toolchains/llvm/prebuilt/linux-x86_64/bin/clang" --version
```

## Rebuild the corresponding decoder from pinned sources

Run from a fresh working directory. You may obtain the FFmpeg source either by checking out the pinned commit, as shown below, **or** by extracting `ffmpeg-source-573f2b81.tar.gz` into the same `jni/ffmpeg` destination. Do not mix different FFmpeg revisions.

```bash
mkdir -p "$HOME/nebula-ffmpeg-rebuild"
cd "$HOME/nebula-ffmpeg-rebuild"

git clone https://github.com/androidx/media.git media3
git -C media3 checkout 8c6678b657ede1e7883fc164ef73ed483c7796c3

FFMPEG_MODULE_PATH="$PWD/media3/libraries/decoder_ffmpeg/src/main"
git clone https://github.com/FFmpeg/FFmpeg.git "$FFMPEG_MODULE_PATH/jni/ffmpeg"
git -C "$FFMPEG_MODULE_PATH/jni/ffmpeg" checkout 573f2b81ebd97da81def2ba762e81b0328547004

cd "$FFMPEG_MODULE_PATH/jni"
./build_ffmpeg.sh "$FFMPEG_MODULE_PATH" "$NDK_PATH" linux-x86_64 23 ac3 eac3

cd "$HOME/nebula-ffmpeg-rebuild/media3"
./gradlew :lib-decoder-ffmpeg:assembleRelease

# The project uses a redirected Gradle build directory: 'buildout'.
find libraries/decoder_ffmpeg -path '*/outputs/aar/*.aar' -type f -print
```

Expected output path for the observed Media3 build setup:

`libraries/decoder_ffmpeg/buildout/outputs/aar/lib-decoder-ffmpeg-release.aar`

The resulting AAR contains `jni/<ABI>/libffmpegJNI.so` for the four ABIs. Builds need not be byte-for-byte identical due to compiler/tool versions and build metadata. Verify the active library and enabled decoder configuration instead of relying only on file names.

## Configuration to verify

The corresponding FFmpeg build used `build_ffmpeg.sh` with Android API `23` and decoders `ac3 eac3`. The observed generated configuration included:

```text
CONFIG_GPL=0
CONFIG_NONFREE=0
CONFIG_VERSION3=0
CONFIG_AC3_DECODER=1
CONFIG_EAC3_DECODER=1
--enable-static --disable-shared --disable-everything --disable-asm
--enable-swresample --enable-decoder=ac3 --enable-decoder=eac3
```

These are **observed checks**, not a replacement for preserving and reviewing the exact generated configure options, patches, and dependency licenses. FFmpeg is statically linked into the `libffmpegJNI.so` JNI wrapper, so the ability to relink that wrapper with a modified LGPL component is relevant.

## Relinking and substituting a modified FFmpeg build

The release's `ffmpeg-relink-materials.tar.gz` contains the twelve `.a` libraries (`libavcodec.a`, `libavutil.a`, `libswresample.a` for each ABI) and four JNI wrapper object files (`ffmpeg_jni.cc.o`). The `.o` files were produced by the Media3 JNI wrapper at the source revision above; they are not a whole Nebula application build.

A practical way to test relinking is to rebuild the FFmpeg libraries from a **modified** copy of the pinned FFmpeg source, then relink the Media3 wrapper:

1. Apply your changes to the FFmpeg source at `"$FFMPEG_MODULE_PATH/jni/ffmpeg"`. Keep the LGPL-compatible build settings.
2. From `"$FFMPEG_MODULE_PATH/jni"`, rerun `./build_ffmpeg.sh "$FFMPEG_MODULE_PATH" "$NDK_PATH" linux-x86_64 23 ac3 eac3`. It regenerates ABI-specific libraries under `jni/ffmpeg/android-libs/<ABI>/`.
3. From the Media3 root, rerun `./gradlew :lib-decoder-ffmpeg:assembleRelease` to build/relink and package the new JNI `.so` files.
4. Confirm the updated `libffmpegJNI.so` for the target ABI is present in the produced AAR with `unzip -Z -1 <path-to-AAR> | grep 'libffmpegJNI.so'`.

For an already-configured build tree with an updated ABI-specific static library, it is also possible to relink the wrapper using CMake. The generated `.cxx` folder contains a **variable build identifier**, so discover it rather than copying the build ID from someone else's machine:

```bash
cd "$HOME/nebula-ffmpeg-rebuild/media3"
MODULE="libraries/decoder_ffmpeg"
ABI="arm64-v8a"
BUILD_DIR="$(find "$MODULE/.cxx/RelWithDebInfo" -mindepth 2 -maxdepth 2 -type d -name "$ABI" -print -quit)"
[ -n "$BUILD_DIR" ] || { echo 'Run the Gradle release build first'; exit 1; }
cmake --build "$BUILD_DIR" --target ffmpegJNI --clean-first
```

**Modified FFmpeg relinking verification (November 9, 2026):** Modified FFmpeg source was successfully rebuilt and relinked for all four Android ABIs. A separate Nebula test APK was installed on a Google Pixel 6a and successfully played E-AC-3 audio. Android Logcat confirmed execution of the modified FFmpeg function through `NEBULA_DECODER_REQUEST=eac3` and `NEBULA_RELINK_TEST_20261108`. Full evidence is documented in [RELINK_VERIFICATION.md](RELINK_VERIFICATION.md). Independent reproduction using only public materials and complete LGPL compliance remain subject to separate verification.

## Distribution notice and remaining checks

Nebula Music Player uses software from the [FFmpeg project](https://ffmpeg.org/) under the GNU Lesser General Public License, version 2.1 or later. The corresponding source and relinking materials are available in the [Nebula FFmpeg release](https://github.com/Nebula-dev-15/nebula-ffmpeg-licenses/releases/tag/ffmpeg-eac3-v1.0). The Nebula application's proprietary source is not part of this FFmpeg source distribution.

Before shipping the application publicly, independently confirm: complete corresponding-source/build material, practical modified-library relinking and replacement, correct license notice and terms (without restrictions on LGPL debugging/reverse engineering), consistency with the AAR actually distributed, and separate E-AC-3 technology/patent rights. FFmpeg's LGPL grant **does not by itself clear Dolby-related patent or trademark rights**. Public release files and this guide are not a legal certification or Play Store approval.

References: [FFmpeg Legal Checklist](https://ffmpeg.org/legal.html), [LGPL 2.1](https://www.gnu.org/licenses/old-licenses/lgpl-2.1.html), [AndroidX Media3 FFmpeg README](https://github.com/androidx/media/blob/release/libraries/decoder_ffmpeg/README.md).
