# onnxruntime-osx-x64

Builds [microsoft/onnxruntime](https://github.com/microsoft/onnxruntime) from source for
**macOS Intel (osx-x64)**, CPU execution provider only, and publishes the result as a GitHub
Release asset.

## Why this exists

Microsoft's own `Microsoft.ML.OnnxRuntime*` NuGet packages stopped shipping osx-x64 (Intel Mac)
native binaries starting at ONNX Runtime 1.24.1, which also raised the minimum macOS version to
14.0. There is no official prebuilt core runtime for Intel Macs at or after that version. This
repo closes that packaging gap for [StreamDock](https://github.com/streamdock-dev), the same way
[`ffmpeg-build-script`](https://github.com/streamdock-dev/ffmpeg-build-script) closes an
equivalent gap for a genuinely LGPL macOS ffmpeg build.

See StreamDock's `Docs/onnx-execution-provider-investigation.md` and
`Docs/Reference/platform-support.md` for the consumer side: how `StreamDock.Build`'s
`OnnxRuntime.targets` picks this up for the osx-x64 RID.

## What gets built

CPU execution provider only, as a single-architecture (`x86_64`) `libonnxruntime.dylib`, with
`CMAKE_OSX_DEPLOYMENT_TARGET` pinned to match StreamDock's own current macOS floor (`13.0` as of
this writing -- see that repo's platform-support doc; bump both together if that floor ever
changes). No GPU execution provider: DirectML is Windows-only and the WebGPU EP plugin doesn't
ship osx-x64 natives either (a possible future addition to this repo, not attempted yet).

## Versioning

Tags look like `v<onnxruntime-version>-streamdock.<n>`, e.g. `v1.24.4-streamdock.1` builds
upstream's `v1.24.4` tag. The `<n>` suffix increments if this repo's own build script changes
without an upstream version bump (a flag tweak, a new verification check). **Must stay in sync
with the `Microsoft.ML.OnnxRuntime`/`.Managed`/`.DirectML` version pinned in StreamDock.Build's
`Sdk.props`** -- a managed assembly newer than the native runtime it's paired with fails to bind
(see the main app's investigation doc, "Finding 2").

A workflow_dispatch input (`onnxruntime_version`) is also available for a one-off build without
cutting a tag, e.g. to sanity-check a new upstream version before committing to it.

## How it's built

GitHub's hosted `macos-15-intel` runner (free for public repos; see
[GitHub's runner image list](https://github.com/actions/runner-images)), running upstream's own
`build.sh` with `--cmake_extra_defines CMAKE_OSX_ARCHITECTURES=x86_64` so one invocation produces
a pure x86_64 slice regardless of the (Intel) host's own architecture. No Homebrew dependency --
onnxruntime builds protobuf/abseil/onnx/re2/nsync from source via its own submodules. The build
job verifies the output's architecture, deployment target, and that nothing links a host-specific
path (`/usr/local`, `/opt/homebrew`, a runner home directory) before publishing -- the same class
of bug `ffmpeg-build-script`'s fork hit on this exact runner family, caught here before release
rather than after.

## License

This repo's own build script: MIT, see `LICENSE`. The published binary is built from
microsoft/onnxruntime's unmodified source at the tagged version; its own MIT license is copied
into the release artifact as `LICENSE.onnxruntime`.
