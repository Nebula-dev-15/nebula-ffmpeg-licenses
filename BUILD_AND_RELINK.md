# FFmpeg Modified-Source Relinking Verification

**Project:** Nebula Music Player  
**Verification date:** November 9, 2026  
**Result:** Successful — modified FFmpeg code executed inside an Android test application.

## 1. Purpose

This document records a practical test demonstrating that a modified FFmpeg component can be rebuilt, relinked into the Media3 FFmpeg JNI library, packaged in a separate Nebula Music Player test APK, and executed on an Android device.

This is technical verification evidence, not a certification of complete LGPL compliance.

## 2. Source Versions

| Component | Version |
|---|---|
| FFmpeg | Commit `573f2b81ebd97da81def2ba762e81b0328547004` |
| AndroidX Media3 | Commit `8c6678b657ede1e7883fc164ef73ed483c7796c3` |
| Media3 version | 1.11.1 |
| Nebula test build | 1.3.8 (version code 17) |
| Android NDK | 28.2.13676358 |
| Minimum Android API | 23 |

Published source and relinking materials:

[FFmpeg LGPL Release — ffmpeg-eac3-v1.0](https://github.com/Nebula-dev-15/nebula-ffmpeg-licenses/releases/tag/ffmpeg-eac3-v1.0)

## 3. Build and Relinking Test

The test was performed using Ubuntu on WSL2, with Linux Android SDK/NDK tools.

The pinned FFmpeg source was built with the `ac3` and `eac3` decoders enabled.

A unique test marker was inserted into FFmpeg's `avcodec_find_decoder_by_name()` function in `libavcodec/allcodecs.c`:

`NEBULA_RELINK_TEST_20261108`

The modified FFmpeg static libraries were rebuilt for four Android architectures:

- `arm64-v8a`
- `armeabi-v7a`
- `x86`
- `x86_64`

The AndroidX Media3 FFmpeg JNI library was then rebuilt and relinked. The test marker was found in the resulting `libffmpegJNI.so` binaries for all four architectures.

For Android runtime verification, the JNI test wrapper was additionally instrumented to log decoder requests and forward relevant FFmpeg logging to Android Logcat.

These logging modifications were made only for verification.

## 4. Android Application Test

A separate debug APK was compiled using the modified FFmpeg JNI library.

| Property | Value |
|---|---|
| Test package | `com.nebula.musicplayer.relinktest` |
| Test version | 1.3.8 (17) |
| Test device | Google Pixel 6a (`bluejay`) |
| Device ABI | `arm64-v8a` |
| Android API | 37 |
| APK compilation | `BUILD SUCCESSFUL` |
| APK installation | `Success` |

The separate test package ensured that the original `com.nebula.musicplayer` installation was not replaced.

Android Logcat confirmed successful loading of `libffmpegJNI.so` from the test application.

## 5. E-AC-3 Playback Verification

The test audio file was inspected with FFprobe:

```text
codec_name=eac3
codec_long_name=ATSC A/52B (AC-3, E-AC-3)
```

The file was played successfully in the separate Nebula test application, with audible audio output confirmed on the Pixel 6a.

### Direct evidence of modified code execution

The following lines were captured from Android Logcat on November 9, 2026:

```text
11-09 00:45:54.682 D/ffmpeg_jni(6629): NEBULA_DECODER_REQUEST=eac3
11-09 00:45:54.683 W/NebulaRelink(6629): NEBULA_RELINK_TEST_20261108

11-09 00:45:54.684 D/ffmpeg_jni(6629): NEBULA_DECODER_REQUEST=eac3
11-09 00:45:54.684 W/NebulaRelink(6629): NEBULA_RELINK_TEST_20261108

11-09 00:45:54.691 D/ffmpeg_jni(6629): NEBULA_DECODER_REQUEST=eac3
11-09 00:45:54.691 W/NebulaRelink(6629): NEBULA_RELINK_TEST_20261108
```

The `NEBULA_DECODER_REQUEST=eac3` entries show the JNI wrapper requesting the E-AC-3 decoder.

The corresponding `NEBULA_RELINK_TEST_20261108` entries confirm that execution reached the modified FFmpeg function.

Together, these observations establish direct evidence that the modified FFmpeg code executed inside the Android test application.

## 6. Binary Verification

SHA-256 hashes recorded for an intermediate modified JNI build, before the additional Android logging instrumentation:

| ABI | SHA-256 |
|---|---|
| arm64-v8a | `c4aaa84e0f7a0c1ca3849ae284482866afe4db3554d1c205872c7528fd384643` |
| armeabi-v7a | `cb176c6d6ac1d6cb4f061bf88b0a6aa5ce8f82e704e6887e63fa8aa664594fee` |
| x86 | `823ac1a6a3b05a7c5e33ce0d86e02afcdc11df905dfb99b81903cc6c88c9ee65` |
| x86_64 | `50d7ca62b82076a711c4b7e30fd1169083c6094daff2c6259ad3f9706c1c3db8` |

These hashes identify that intermediate build only. They are not hashes of the final, additionally instrumented test APK or the public release artifacts.

## 7. Conclusion and Limitations

**Verified successfully:**

- Rebuilding FFmpeg after modifying its source.
- Relinking modified FFmpeg into Media3 JNI libraries for four Android ABIs.
- Packaging the modified JNI library into a separate Android application.
- Installing and loading that library on a physical Android device.
- Playing an E-AC-3 audio file.
- Directly observing execution of the modified FFmpeg function through Android Logcat.

**Limitations:**

- Runtime execution was tested on `arm64-v8a`; the other three ABIs were verified through compilation, relinking, and binary inspection.
- The test used a locally available Nebula source tree. Independent end-user reconstruction using only the publicly released materials has not been separately verified.
- This verification does not establish that all corresponding-source, relinking-material, license-notice, or reverse-engineering requirements have been satisfied.
- The verification-specific logging code is not intended for production distribution.
- FFmpeg's LGPL does not independently establish E-AC-3 patent or other technology licensing rights.

The test demonstrates successful modified-source FFmpeg relinking and runtime execution within a Nebula Music Player test application.

For rebuild instructions, see [BUILD_AND_RELINK.md](https://github.com/Nebula-dev-15/nebula-ffmpeg-licenses/blob/main/BUILD_AND_RELINK.md).
