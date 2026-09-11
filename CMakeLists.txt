cmake_minimum_required(VERSION 3.16)
project(webrtc_apm LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# ============================================================
# 1. 平台 & 编译器检测，设置宏和编译选项
# ============================================================

if(WIN32)
    add_compile_definitions(WEBRTC_WIN NOMINMAX WIN32_LEAN_AND_MEAN
_CRT_SECURE_NO_WARNINGS _USE_MATH_DEFINES)
    if(MSVC)
        add_compile_options(/W0 /EHsc /utf-8 /wd4996)
    else()
        add_compile_options(-w -fPIC)
        add_link_options("-Wl,--allow-multiple-definition")
    endif()
elseif(ANDROID)
    add_compile_definitions(WEBRTC_ANDROID WEBRTC_POSIX
                           WEBRTC_ANDROID_PLATFORM_BUILD)
    add_compile_options(-w -fPIC -pthread)
elseif(APPLE)
    add_compile_definitions(WEBRTC_APPLE WEBRTC_MAC WEBRTC_POSIX)
    if(CMAKE_SYSTEM_NAME STREQUAL "iOS" OR CMAKE_OSX_SYSROOT MATCHES "iphone")
        add_compile_definitions(WEBRTC_IOS)
    else()
        add_compile_definitions(WEBRTC_OSX)
    endif()
    add_compile_options(-w -fPIC -pthread)
elseif(UNIX)
    add_compile_definitions(WEBRTC_LINUX WEBRTC_POSIX)
    add_compile_options(-w -fPIC -pthread)
elseif(EMSCRIPTEN)
    add_compile_definitions(WEBRTC_LINUX WEBRTC_POSIX)
    add_compile_options(-w -fPIC)
else()
    message(FATAL_ERROR "Unsupported platform: ${CMAKE_SYSTEM_NAME}")
endif()

# 架构宏 —— 不手动定义，让 rtc_base/system/arch.h 自动检测
# （它会用 _M_X64 / __x86_64__ / __aarch64__ 等编译器预定义宏）

# 通用宏
add_compile_definitions(
    WEBRTC_EXCLUDE_TRANSIENT_SUPPRESSOR
    WEBRTC_APM_DEBUG_DUMP=0
    RTC_ENABLE_VOICE_ENGINE_AUDIO_PROCESSING=1
    RTC_USE_SYSTEM_CONSOLE=1
)

include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/third_party/abseil-cpp
)

# ============================================================
# 2. CMake 层面架构检测（用于源码收集时过滤架构特定汇编文件）
#    注意：只用于 CMake 变量；C++ 侧仍由 rtc_base/system/arch.h 自动检测
# ============================================================
if(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64|AMD64|amd64|X86")
    set(WEBRTC_ARCH_X86_FAMILY TRUE)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64|arm64|ARM64|armv7|armv8")
    set(WEBRTC_ARCH_ARM_FAMILY TRUE)
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "mips|MIPS")
    set(WEBRTC_ARCH_MIPS_FAMILY TRUE)
else()
    # 保守处理：不排除任何架构特定文件，让编译时自行处理
endif()

# ============================================================
# 3. collect_sources —— 按目录收集 .cc/.c 文件，过滤测试/工具
# ============================================================

function(collect_sources out_var)
    set(_result "")
    foreach(_dir ${ARGN})
        message(STATUS "COLLECT_DIR: ${_dir}")
        file(GLOB_RECURSE _srcs CONFIGURE_DEPENDS
            "${CMAKE_CURRENT_SOURCE_DIR}/${_dir}/*.cc"
            "${CMAKE_CURRENT_SOURCE_DIR}/${_dir}/*.c"
        )
        foreach(_src ${_srcs})
            get_filename_component(_name ${_src} NAME)
            get_filename_component(_basename ${_src} NAME_WE)
            set(_relpath "${_src}")
            string(REPLACE "${CMAKE_CURRENT_SOURCE_DIR}/" "" _relpath "${_relpath}")

            if(_relpath MATCHES "/mock/" OR _relpath MATCHES "/aec_dump/")
                continue()
            endif()
            if(_name MATCHES "unittest|_test\\.|test_|benchmark")
                continue()
            endif()

            if(_basename STREQUAL "click_annotate" OR _basename STREQUAL "rnn_vad_tool" OR _basename STREQUAL "apm_data_dumper_tools")
                continue()
            endif()

            if(_relpath MATCHES "/video_coding/" OR
               _relpath MATCHES "/video_processing/" OR _relpath MATCHES "/rtp_rtcp/" OR
               _relpath MATCHES "/remote_bitrate_estimator/" OR _relpath MATCHES "/congestion_controller/" OR
               _relpath MATCHES "/pacing/")
                continue()
            endif()

            if(_name MATCHES "_neon\\." OR _relpath MATCHES "/neon/")
                if(NOT WEBRTC_ARCH_ARM_FAMILY)
                    continue()
                endif()
            elseif(_name MATCHES "_mips\\." OR _relpath MATCHES "/mips/")
                if(NOT WEBRTC_ARCH_MIPS_FAMILY)
                    continue()
                endif()
            elseif(_name MATCHES "_sse4_1\\." OR _relpath MATCHES "/sse4_1/" OR
                   _name MATCHES "_avx2\\." OR _relpath MATCHES "/avx2/")
                if(NOT WEBRTC_ARCH_X86_FAMILY)
                    continue()
                endif()
            endif()

            list(APPEND _result ${_src})
        endforeach()
    endforeach()

    set(${out_var} ${_result} PARENT_SCOPE)
endfunction()

# ============================================================
# 4. 收集所有源文件
# ============================================================

collect_sources(APM_SOURCES
    "modules/audio_processing"
    "api/audio"
    "common_audio"
    "system_wrappers/source"
    "api/task_queue"
    "modules/audio_coding/codecs/isac/main/source"
    "modules/audio_coding/codecs/isac/main/util"
    "modules/audio_coding/codecs/isac/fix/source"
)

collect_sources(RTC_SOURCES
    "rtc_base"
    "third_party/pffft/src"
    "third_party/rnnoise/src"
)

# ============================================================
# 5. 排除规则
#    —— 简单的文件名 + 路径子串匹配，避免正则复杂性
# ============================================================

set(_exclude_basenames
    "rtp_to_ntp_estimator"
    "echo_canceller3_factory" "echo_detector_creator"
    "ssl_adapter" "ssl_certificate" "ssl_fingerprint" "ssl_identity" "ssl_stream_adapter"
    "openssl_adapter" "openssl_certificate" "openssl_digest" "openssl_identity"
    "openssl_session_cache" "openssl_stream_adapter" "openssl_utility"
    "fake_ssl_identity" "rtc_certificate" "rtc_certificate_generator"
    "json" "echo_canceller3_config_json"
    "thread_registry" "warn_current_thread_is_deadlocked"
    "gunit" "helpers" "message_digest" "mac_ifaddrs_converter"
    "alr_experiment" "balanced_degradation_settings" "cpu_speed_experiment"
    "field_trial_list" "field_trial_units" "struct_parameters_parser"
    "jitter_upper_bound_experiment" "keyframe_interval_settings"
    "min_video_bitrate_experiment" "normalize_simulcast_size_experiment"
    "quality_rampup_experiment" "quality_scaler_settings" "quality_scaling_experiment"
    "rate_control_settings" "rtt_mult_experiment"     "stable_target_rate_experiment"
    "sent_packet" "fifo_buffer"
    "event_based_exponential_moving_average" "event_rate_counter" "exp_filter"
    "histogram_percentile_counter" "samples_stats_counter" "sample_stats"
    "audio_format_to_string" "string_format"
    "pending_task_safety_flag" "repeating_task"
    "base64" "sigslot"
    "timestamp_extrapolator" "windows_version"
        "sinusoidal_linear_chirp_source"
    "cross_correlation_mips" "cross_correlation_neon"
    "auto_corr_to_refl_coef"
    "filter_ar" "filter_ar_fast_q12" "filter_ar_fast_q12_mips"
    "filter_ma_fast_q12"
    "ilbc_specific_functions" "levinson_durbin" "lpc_to_refl_coef"
        "refl_coef_to_lpc"
    "audio_decoder_isac" "audio_encoder_isac"
    "audio_decoder_isacfix" "audio_encoder_isacfix"
)

set(_exclude_path_substrings
    "/modules/audio_processing/transient/"
    "/audio/utility/"
    "/third_party/abseil-cpp/"
    "/java/"
)

if(WIN32)
    list(APPEND _exclude_basenames
        "rw_lock_posix" "task_queue_gcd" "task_queue_libevent"
        "task_queue_stdlib" "task_queue_posix"
        "ifaddrs_android" "ifaddrs_converter" "mac_ifaddrs"
        "default_task_queue_factory_stdlib" "default_task_queue_factory_gcd"
        "default_task_queue_factory_libevent"
        "cpu_features_linux" "cpu_features_android"
    )
elseif(ANDROID)
    list(APPEND _exclude_basenames
        "rw_lock_win" "task_queue_win"
        "task_queue_gcd" "task_queue_libevent"
        "ifaddrs_converter" "mac_ifaddrs"
        "default_task_queue_factory_win" "default_task_queue_factory_gcd"
        "default_task_queue_factory_libevent"
        "cpu_features_linux"
    )
elseif(APPLE)
    list(APPEND _exclude_basenames
        "rw_lock_win" "task_queue_win"
        "ifaddrs_android" "ifaddrs_converter"
        "default_task_queue_factory_win" "default_task_queue_factory_stdlib"
        "default_task_queue_factory_libevent"
        "cpu_features_linux" "cpu_features_android"
    )
elseif(UNIX)
    list(APPEND _exclude_basenames
        "rw_lock_win" "task_queue_win"
        "ifaddrs_android" "mac_ifaddrs"
        "default_task_queue_factory_win" "default_task_queue_factory_gcd"
        "default_task_queue_factory_libevent"
        "cpu_features_android"
    )
endif()

function(apply_excludes out_var _srcs)
    set(_filtered "")
    foreach(_src ${_srcs})
        set(_exclude FALSE)
        get_filename_component(_bn ${_src} NAME_WE)
        list(FIND _exclude_basenames "${_bn}" _idx)
        if(_idx GREATER -1)
            set(_exclude TRUE)
        endif()
        if(NOT _exclude)
            foreach(_sub ${_exclude_path_substrings})
                string(FIND "${_src}" "${_sub}" _pos)
                if(_pos GREATER -1)
                    set(_exclude TRUE)
                    break()
                endif()
            endforeach()
        endif()
        if(NOT _exclude)
            list(APPEND _filtered ${_src})
        endif()
    endforeach()
    set(${out_var} ${_filtered} PARENT_SCOPE)
endfunction()

apply_excludes(_filtered_apm "${APM_SOURCES}")

apply_excludes(_filtered_rtc "${RTC_SOURCES}")

# 移除 APM_SOURCES 中已废弃的 header-only 子模块
set(_filtered_apm_clean "")
foreach(_src ${_filtered_apm})
    get_filename_component(_wname ${_src} NAME_WE)
    if(_wname STREQUAL "fixed_gain_controller")
        continue()
    endif()
    list(APPEND _filtered_apm_clean ${_src})
endforeach()
set(_filtered_apm ${_filtered_apm_clean})


# ============================================================
# 6. 显式追加 third_party 关键源文件（保证必被编译）
# ============================================================

set(THIRD_PARTY_SOURCES
    third_party/pffft/src/pffft.c
    third_party/pffft/src/fftpack.c
    third_party/rnnoise/src/rnn_vad_weights.cc
    common_audio/third_party/spl_sqrt_floor/spl_sqrt_floor.c
    common_audio/third_party/ooura/fft_size_256/fft4g.cc
    common_audio/third_party/ooura/fft_size_128/ooura_fft.cc
    modules/audio_processing/aec_dump/null_aec_dump_factory.cc
)
# ============================================================
# 7. 构建静态库
# ============================================================

add_library(webrtc_apm STATIC
    ${_filtered_apm}
    ${_filtered_rtc}
    ${THIRD_PARTY_SOURCES}
)

target_include_directories(webrtc_apm PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/third_party/abseil-cpp
)

if(WIN32)
    target_link_libraries(webrtc_apm PUBLIC winmm ws2_32)
elseif(ANDROID)
    target_link_libraries(webrtc_apm PUBLIC pthread log)
elseif(UNIX)
    target_link_libraries(webrtc_apm PUBLIC pthread)
endif()

set_target_properties(webrtc_apm PROPERTIES OUTPUT_NAME "webrtc_apm")

# ============================================================
# Example 可执行文件（不在 ALL 目标中，需要显式 --target）
# ============================================================

option(APM_BUILD_EXAMPLES "Build example executables" ON)

if(APM_BUILD_EXAMPLES)
    add_executable(basic_apm example/basic_apm.cc)
    target_link_libraries(basic_apm PRIVATE webrtc_apm)
    set_target_properties(basic_apm PROPERTIES RUNTIME_OUTPUT_DIRECTORY
        ${CMAKE_BINARY_DIR}/examples)

    add_executable(apm_pipeline example/apm_pipeline.cc)
    target_link_libraries(apm_pipeline PRIVATE webrtc_apm)
    set_target_properties(apm_pipeline PROPERTIES RUNTIME_OUTPUT_DIRECTORY
        ${CMAKE_BINARY_DIR}/examples)

    add_executable(aec_demo example/aec_demo.cc)
    target_link_libraries(aec_demo PRIVATE webrtc_apm)
    set_target_properties(aec_demo PROPERTIES RUNTIME_OUTPUT_DIRECTORY
        ${CMAKE_BINARY_DIR}/examples)

    add_executable(ns_demo example/ns_demo.cc)
    target_link_libraries(ns_demo PRIVATE webrtc_apm)
    set_target_properties(ns_demo PROPERTIES RUNTIME_OUTPUT_DIRECTORY
        ${CMAKE_BINARY_DIR}/examples)

    add_executable(agc_demo example/agc_demo.cc)
    target_link_libraries(agc_demo PRIVATE webrtc_apm)
    set_target_properties(agc_demo PROPERTIES RUNTIME_OUTPUT_DIRECTORY
        ${CMAKE_BINARY_DIR}/examples)

    add_executable(vad_demo example/vad_demo.cc)
    target_link_libraries(vad_demo PRIVATE webrtc_apm)
    set_target_properties(vad_demo PROPERTIES RUNTIME_OUTPUT_DIRECTORY
        ${CMAKE_BINARY_DIR}/examples)

    add_executable(hpf_aec_ns_agc_vad example/hpf_aec_ns_agc_vad.cc)
    target_link_libraries(hpf_aec_ns_agc_vad PRIVATE webrtc_apm)
    set_target_properties(hpf_aec_ns_agc_vad PROPERTIES RUNTIME_OUTPUT_DIRECTORY
        ${CMAKE_BINARY_DIR}/examples)
endif()

list(LENGTH _filtered_apm _apm_count)
list(LENGTH _filtered_rtc _rtc_count)

message(STATUS "")
message(STATUS "=== WebRTC APM Build Summary ===")
message(STATUS "  Platform : ${CMAKE_SYSTEM_NAME}")
message(STATUS "  Compiler : ${CMAKE_CXX_COMPILER_ID} ${CMAKE_CXX_COMPILER_VERSION}")
message(STATUS "  Arch     : ${CMAKE_SYSTEM_PROCESSOR}")
message(STATUS "  Config   : ${CMAKE_BUILD_TYPE}")
message(STATUS "  Sources  : APM=${_apm_count}, RTC=${_rtc_count}")
message(STATUS "  Output   : webrtc_apm${CMAKE_STATIC_LIBRARY_SUFFIX}")
message(STATUS "================================")
message(STATUS "")