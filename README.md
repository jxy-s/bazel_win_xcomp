# Bazel Windows Cross-Compile

This repo demonstrates how to cross-compile a simple C++ program for Windows using Bazel. The scope
of this subject is compiling for a Windows target from a Windows host that is not the host native
architecture. This is normally possible using the `--platforms` flag in Bazel by specifying the
constraints for the target platform.

## ATTENTION

There appears to be a bug in toolchain resolution for bazel right now. There are two observed
problems:

1. `--platform` for a target other than the host fails to resolve the toolchain.
2. On an ARM64 Windows host the output binary is x64 when no platform target is specified.

The consequence of this is that the toolchain resolution is incapable of producing a binary for
anything other than x64. Counterintuitively, the ARM64 host produces an x64 binary

I've tested this on my Windows x64 and ARM64 machines. Both have all the MSVC installations
necessary to cross-compile to another architecture:

```shell
➜ Get-ChildItem -Include cl.exe -Recurse "C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools" | ForEach-Object{echo $_.FullName}
C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.39.33519\bin\Hostx64\arm\cl.exe
C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.39.33519\bin\Hostx64\arm64\cl.exe
C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.39.33519\bin\Hostx64\x64\cl.exe
C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.39.33519\bin\Hostx64\x86\cl.exe
C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.39.33519\bin\Hostx86\arm\cl.exe
C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.39.33519\bin\Hostx86\arm64\cl.exe
C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.39.33519\bin\Hostx86\x64\cl.exe
C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.39.33519\bin\Hostx86\x86\cl.exe

➜ Get-ChildItem -Include vcvar*.bat -Recurse "C:\Program Files\Microsoft Visual Studio\2022\Community\VC" | ForEach-Object{echo $_.FullName}
C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvars32.bat
C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvars64.bat
C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvarsall.bat
C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvarsamd64_arm.bat
C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvarsamd64_arm64.bat
C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvarsamd64_x86.bat
C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvarsx86_amd64.bat
C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvarsx86_arm.bat
C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvarsx86_arm64.bat
```

```shell
➜ bazel build //... --platforms=:windows_arm64 --toolchain_resolution_debug=.*
WARNING: Build option --toolchain_resolution_debug has changed, discarding analysis cache (this can be expensive, see https://bazel.build/advanced/performance/iteration-speed).
INFO: ToolchainResolution: Target platform //:windows_arm64: Selected execution platform @@local_config_platform//:host,
INFO: ToolchainResolution: Performing resolution of @@bazel_tools//tools/cpp:toolchain_type for target platform //:windows_arm64
      ToolchainResolution:   Rejected toolchain @@bazel_tools~cc_configure_extension~local_config_cc//:cc-compiler-armeabi-v7a; mismatching values: armv7, android
      ToolchainResolution:   Rejected toolchain @@bazel_tools~cc_configure_extension~local_config_cc//:cc-compiler-x64_windows; mismatching values: x86_64
      ToolchainResolution:   Rejected toolchain @@bazel_tools~cc_configure_extension~local_config_cc//:cc-compiler-armeabi-v7a; mismatching values: armv7, android
      ToolchainResolution:   Rejected toolchain @@bazel_tools~cc_configure_extension~local_config_cc//:cc-compiler-x64_windows; mismatching values: x86_64
      ToolchainResolution:   Rejected toolchain @@local_config_cc//:cc-compiler-armeabi-v7a; mismatching values: armv7, android
      ToolchainResolution:   Rejected toolchain @@local_config_cc//:cc-compiler-x64_windows; mismatching values: x86_64
      ToolchainResolution: No @@bazel_tools//tools/cpp:toolchain_type toolchain found for target platform //:windows_arm64.
INFO: ToolchainResolution: Target platform //:windows_arm64: Selected execution platform @@local_config_platform//:host,
ERROR: C:/users/jxy/_bazel_jxy/veelwacq/external/bazel_tools/tools/cpp/BUILD:58:19: in cc_toolchain_alias rule @@bazel_tools//tools/cpp:current_cc_toolchain:
Traceback (most recent call last):
        File "/virtual_builtins_bzl/common/cc/cc_toolchain_alias.bzl", line 26, column 48, in _impl
        File "/virtual_builtins_bzl/common/cc/cc_helper.bzl", line 219, column 17, in _find_cpp_toolchain
Error in fail: Unable to find a CC toolchain using toolchain resolution. Target: @@bazel_tools//tools/cpp:current_cc_toolchain, Platform: @@//:windows_arm64, Exec platform: @@local_config_platform//:host
ERROR: C:/users/jxy/_bazel_jxy/veelwacq/external/bazel_tools/tools/cpp/BUILD:58:19: Analysis of target '@@bazel_tools//tools/cpp:current_cc_toolchain' failed
ERROR: Analysis of target '//source:example' failed; build aborted: Analysis failed
INFO: Elapsed time: 0.226s, Critical Path: 0.01s
INFO: 1 process: 1 internal.
ERROR: Build did NOT complete successfully
```

### Workaround

The only known workaround I have found is to use an alternate toolchain resolution soluation that
requires the EWDK (https://github.com/0xf005ba11/bazel_ewdk_cc). I would expect bazel to be capable
of this out of the box. Or at very least produce a native binary on the ARM64 host.
