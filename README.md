# WebRTC APM — Stand‑Alone Audio Processing Library

An independent **Audio Processing Module** extracted from WebRTC source code. Only audio‑related features are retained; video, network, ICE, media and other irrelevant modules are stripped out. It provides a **single CMake project, zero external dependencies, cross‑platform** compilation.

---

## Feature List



| Module | Class | Description |
| --- | --- | --- |
| **HPF** | Built‑in within `AudioProcessing` | High‑pass filter for low‑frequency noise suppression (default 100 Hz) |
| **AEC3** | Built‑in within `AudioProcessing` | Third‑generation acoustic echo canceller (frequency‑domain adaptive filtering) |
| **AECM** | Built‑in within `AudioProcessing` | Light‑weight mobile‑oriented echo cancellation (native Android / iOS path) |
| **NS** | Built‑in within `AudioProcessing` | Noise suppression (combined time‑domain + frequency‑domain algorithms) |
| **NSX** | Built‑in within `AudioProcessing` | Deep‑neural‑network‑driven noise suppression powered by RNNoise |
| **AGC1** | Built‑in within `AudioProcessing` | Classic automatic gain control |
| **AGC2** | Built‑in within `AudioProcessing` | Loudness‑based second‑generation AGC |
| **VAD** | Built‑in within `AudioProcessing` | Voice activity detection (dual‑engine: GMM + deep neural network) |
| **HWA** | Built‑in within `AudioProcessing` | Proxy for hardware‑backed AGC / Echo / Suppression (optional) |

Processing pipeline:

```
Capture → HPF → AEC3/AECM → NS/NSX → AGC1/AGC2 → VAD → Output
```

---

## Build Requirements



| Item | Minimum Version |
| --- | --- |
| CMake | 3.16+ |
| C++ Compiler | C++17 capable (MSVC 2019+ / GCC 9+ / Clang 10+) |
| Python | Not required |
| GN / Ninja | Not required |

**Zero external dependencies** —‑‑‑ all source code (including ISAC codec, PFFFT, RNNoise, Abseil‑CPP subset) is built‑in.

---

## Quick Start

### Windows (VS2022)

```
cmake -S . -B build -G "Visual Studio 17 2022" -A x64
cmake --build build --config Release
```

### Linux / macOS

```
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j$(nproc)
```

### Android (NDK)

```
cmake -S . -B build-android \
  -DCMAKE_TOOLCHAIN_FILE=$NDK/build/cmake/android.toolchain.cmake \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_PLATFORM=android-24
cmake --build build-android -j
```

### iOS (Xcode)

```
cmake -S . -B build-ios \
  -G Xcode \
  -DCMAKE_OSX_SYSROOT=iphoneos \
  -DCMAKE_OSX_ARCHITECTURES=arm64
cmake --build build-ios --config Release
```

### Emscripten / WASM

```
emcmake cmake -S . -B build-wasm -DCMAKE_BUILD_TYPE=Release
cmake --build build-wasm -j
```

### Disable example programs

```
cmake -S . -B build -DAPM_BUILD_EXAMPLES=OFF
```

Build artifacts:

```
build/
├── Release/webrtc_apm.lib       # Windows
├── libwebrtc_apm.a              # Linux / macOS / Android / iOS
└── examples/                    # 7 example executables
```

---

## Directory Layout

```
apm/
├── CMakeLists.txt                   # Sole build entry point
│
├── modules/
│   └── audio_processing/
│       ├── include/                 # ★ Public main API
│       │   ├── audio_processing.h          ← create instance / process frames / configure runtime
│       │   ├── config.h                    ← RuntimeConfig (toggle individual modules)
│       │   ├── audio_processing_statistics.h  ← return type of getStatistics()
│       │   ├── audio_frame_proxies.h       ← wrap external AudioFrame
│       │   └── audio_frame_view.h          ← view‑style frame type
│       ├── aec3/                    # AEC3 core implementation
│       ├── aecm/                    # AECM core implementation
│       ├── agc/                     # AGC1
│       ├── agc2/                    # AGC2
│       ├── ns/                      # NS + NSX
│       ├── vad/                     # VAD
│       ├── echo_detector/           # AEC3 helper echo detector
│       ├── transient/               # Transient suppression (excluded; only HPF/AEC/NS/AGC/VAD remain)
│       ├── utility/                 # Helper utilities
│       └── aec_dump/                # null_aec_dump (no I‑O dependency)
│
├── modules/audio_coding/codecs/isac/   # ISAC codec (required by VAD + AEC)
│   ├── main/{source,util}/
│   └── fix/source/
│
├── api/
│   ├── audio/                       # AudioFrame / EchoCanceller3Config / NoiseSuppression / GainControl
│   ├── task_queue/                  # TaskQueue interface (stdlib‑based default implementation)
│   ├── units/                       # Timestamp / TimeDelta / DataRate
│   ├── array_view.h                 # span‑like view type
│   ├── audio_options.h
│   ├── function_view.h
│   ├── ref_counted_base.h
│   ├── scoped_refptr.h
│   └── rtp_headers.h / rtp_packet_info.h / rtp_packet_infos.h
│   ⚠ api/video/ removed —‑‑‑ VideoRotation / VideoCodecType etc are inlined placeholders inside rtp_headers.h
│
├── rtc_base/                        # Trimmed‑down base library (9 sub‑dirs / 242 files)
│   ├── system/                      # arch.h (auto‑detect x86/ARM/MIPS) / cpu_features / …
│   ├── synchronization/             # mutex / rw_lock (win + posix) / critical_section
│   ├── memory/                      # memory helpers
│   ├── task_queue*.cc               # four back‑ends: win / stdlib / gcd / libevent
│   ├── time_utils / clock / sleep   # cross‑platform time primitives
│   ├── system_wrappers/             # field_trial / metrics / cpu_info
│   └── experiments/                 # field‑trial parsing (video‑related references commented‑out)
│
├── common_audio/                    # Signal‑processing support for AEC3/N
│   ├── signal_processing/           # SPL library (resample / fft / filter)
│   ├── third_party/                 # spl_sqrt_floor / ooura_fft
│   └── …
│
├── system_wrappers/                 # clock / field_trial / metrics / cpu_features
│
├── third_party/
│   ├── abseil‑cpp/                  # Abseil‑CPP subset (9 directories / 197 files)
│   │                                # Only: base / algorithm / container /
│   │                                # flags / memory / meta / strings / types / utility
│   ├── pffft/                       # PFFFT single‑precision FFT (2 C source files)
│   └── rnnoise/                     # RNNoise neural‑net VAD (weights + inference)
│
└── example/                         # 7 demonstration programs
    ├── basic_apm.cc                 # Minimal build + instance creation
    ├── apm_pipeline.cc              # Full HPF→AEC→NS→AGC→VAD pipeline
    ├── hpf_aec_ns_agc_vad.cc        # Same pipeline plus statistic output
    ├── aec_demo.cc                  # Stand‑alone AEC demo (ERLE printed)
    ├── ns_demo.cc                   # Stand‑alone NS demo (noise‑reduction dB printed)
    ├── agc_demo.cc                  # Stand‑alone AGC demo (input/output level comparison)
    └── vad_demo.cc                  # Stand‑alone VAD demo (per‑frame speech probability output)
```

---

## Quick API Reference

```
#include "modules/audio_processing/include/audio_processing.h"
#include "api/audio/audio_frame.h"

// 1. Create APM instance
apm::AudioProcessingBuilder builder;
builder.SetCapturePostProcessing(std::make_unique<apm::EchoCanceller3Factory>());
auto apm = builder.Create();

// 2. Initialize: 10 ms frames, 16 kHz, mono, full‑duplex
apm::AudioProcessing::Config cfg;
cfg.echo_canceller.enabled = true;
cfg.noise_suppression.enabled = true;
cfg.gain_controller1.enabled = true;
cfg.gain_controller2.enabled = false;
cfg.high_pass_filter.enabled = true;

apm->ApplyConfig(cfg);
apm->Initialize(16000, 16000, 1, 1, apm::AudioProcessing::kFullDuplex);

// 3. Process each 10‑ms audio frame
//    capture_frame: microphone input
//    render_frame:  far‑end reference (mandatory for AEC)
apm->ProcessStream(capture_frame);
apm->ProcessReverseStream(render_frame);

// 4. Read runtime statistics
auto stats = apm->GetStatistics();
// stats.echo_cancel.erl_db      — echo‑return‑loss in dB
// stats.noise_suppression.noise_level — noise‑level estimate
// stats.voice_detected          — VAD decision result

// 5. Clean‑up
apm.reset();
```

Full API documentation can be found in `modules/audio_processing/include/audio_processing.h`.

---

## Example Programs

Compiled binaries are placed under `build/examples/` or `build‑wasm/examples/`.
They take no command‑line arguments; each generates synthetic test audio and applies processing.



| Example | Output Metrics |
| --- | --- |
| `basic_apm` | Instance creation + single‑frame processing success flag |
| `apm_pipeline` | Full HPF→AEC→NS→AGC→VAD pipeline run |
| `hpf_aec_ns_agc_vad` | Pipeline plus ERLE / noise‑reduction dB / VAD probability |
| `aec_demo` | ERLE (dB); simulated echo‑cancellation measurement |
| `ns_demo` | Noise‑reduction amount (dB) |
| `agc_demo` | Input‑level vs output‑level comparison |
| `vad_demo` | Per‑frame speech probability ranging from 0.0 to 1.0 |

---

## Cross‑Platform Support



| Platform | Defines | Linked Libraries | Source‑File Count |
| --- | --- | --- | --- |
| **Windows** | `WEBRTC_WIN` + `NOMINMAX` | `winmm ws2_32` | APM=262 / RTC=42 |
| **Linux** | `WEBRTC_LINUX` + `WEBRTC_POSIX` | `pthread` | same as Windows |
| **macOS** | `WEBRTC_APPLE` + `WEBRTC_OSX` + `WEBRTC_POSIX` | `pthread` | same as Windows |
| **iOS** | `WEBRTC_APPLE` + `WEBRTC_IOS` + `WEBRTC_POSIX` | `pthread` | same as Windows |
| **Android** | `WEBRTC_ANDROID` + `WEBRTC_POSIX` | `pthread log` | uses `cpu_features_android.c`, `ifaddrs_android.cc` |
| **Emscripten** | `WEBRTC_LINUX` + `WEBRTC_POSIX` | (none) | same as Linux |

Automatic architecture detection (`rtc_base/system/arch.h`):

- x86 / x86_64 → enable `_sse4_1` / `_avx2` optimizations
- ARM32 / ARM64 → enable `_neon` optimizations
- MIPS → enable `_mips` optimizations

CMake automatically filters assembly sources for the target architecture according to `CMAKE_SYSTEM_PROCESSOR`.

---

## Refactoring & Minification History

### From original WebRTC source to standalone APM



| Step | Action | Outcome |
| --- | --- | --- |
| 1 | Physically remove 36 unrelated directories: `audio_mixer` / `api/video_codecs` / `api/video` / `call` / `media` / `rtc` / `java` / `video_coding` / `video_processing` / `rtp_rtcp` / `congestion_controller` / `pacing` / `remote_bitrate_estimator` … | ~70 % of top‑level directories removed |
| 2 | CMake path exclusion for `/transient/` / `/audio/utility/` (transient suppression dropped) | Source‑file count: 351 → 262 |
| 3 | Delete all `.gn` / `.gni` / `.m` / BUILD.gn build files | Project build system reduced to pure CMake |
| 4 | Manually exclude 28 irrelevant `.cc` files: `*_factory` / `*_json` / `ssl_*` / `openssl_*` / `*_experiment` / `*_stats_counter` | Further code‑base shrink |
| 5 | **Abseil‑CPP closure trimming**: 18 sub‑dirs / 1259 files → 9 sub‑dirs / 197 files (remove time / random / debugging / flags / protocolbuffers …) | 84.4 % reduction |
| 6 | **rtc_base closure trimming**: 12 sub‑dirs / 509 files → 9 sub‑dirs / 242 files (remove java / network / time / raw win‑platform sources: 157 files) | 52.5 % reduction |
| 7 | **Remove api/video/** directory (29 files) + comment out video‑related includes inside rtp_headers.h + comment video‑type references in rtc_base/experiments | Full removal of video dependencies |

### Build‑Artifact Comparison



| Metric | Original WebRTC | Trimmed APM | Delta |
| --- | --- | --- | --- |
| Compiled source‑file count | ~1500+ | **304** | ↓ 80 % |
| Directory count | ~120 | **55** | ↓ 54 % |
| webrtc_apm.lib | — | **7.36 MB** | — |
| External dependencies | GN / Ninja / Python / OpenSSL … | **Zero** | — |

---

## License

BSD 3‑Clause License, consistent with original WebRTC code‑base.
