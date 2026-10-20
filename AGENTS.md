# AGENTS.md — Throne

Qt6/C++20 desktop proxy GUI (sing-box core); Go core lives in `core/server`. Cross-platform, CMake ≥3.20.

## Build (pwsh; Windows focus — repo build scripts are bash)
- Deps: Qt 6.2–6.11 (set `$env:CMAKE_PREFIX_PATH` to its cmake dir), Ninja, CMake.
- GUI configure+build: `cmake -GNinja -DCMAKE_BUILD_TYPE=RelWithDebInfo -S . -B build` then `cmake --build build`
- Set `$env:INPUT_VERSION` before configure (version baked via `cmake/NkrVersion.h.in`); CMakeLists pins MSVC static runtime + Unity build.
- Core: `$env:GOOS="windows"; $env:GOARCH="amd64"; bash script/build_go.sh` (needs Go 1.26 + protoc; run from repo root; regenerates `core/server/gen/libcore.proto`).

## Test / format
- No C++ tests in-repo (3rdparty gtest is OFF). Go core: `cd core/server; go test ./...`
- Format: `bash script/format_cpp.sh` (clang-format over tracked `.cpp/.h`, skips `3rdparty/`; no root `.clang-format`, so clang-format defaults apply).
- Translations: `bash script/translate.sh` — TS files `zh_CN`, `fa_IR`, `ru_RU`.

## Conventions
- `src/` mirrors `include/`; add every new source/header explicitly to `CMakeLists.txt` (no globbing).
- PascalCase namespaces (`Configs`, `Stats`, `Sys`), `#pragma once` headers, `dialog_*.cpp` for dialogs, `.ui` files alongside `.h`.
- Don't edit `3rdparty/` (vendored Qv2ray/SQLiteCpp/simple-protobuf/quirc/Hotkey).

## Release (needs confirmation)
- CI release: `.github/workflows/build.yml` (workflow_dispatch). Packaging scripts: `script/deploy_windows.sh`, `deploy_linux64.sh`, `deploy_macos.sh`, `pack_debian.sh`, `windows_installer.nsi`.
