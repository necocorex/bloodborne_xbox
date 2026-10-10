# Xbox D3D12.x Port: Investigation and Migration Plan

Status: design checkpoint; no Xbox-native renderer is implemented by this document.

## Goal

Move the `bloodborne_xbox` fork toward a native Xbox Series X|S Dev Mode graphics path using the D3D12.x API surface available to the app. Keep the existing desktop Vulkan path intact until the Xbox path can render and present a frame.

The reported Dev Mode process-memory ceiling is approximately 5 GiB. Treat that as a hard package/runtime constraint, not as a target allocation for the game. Measure actual working headroom on-device.

## Current source-level blockers

The current `gpu/CMakeLists.txt` is explicitly Vulkan-centric:

- `find_package(Vulkan REQUIRED)`
- Vulkan Memory Allocator (VMA) is required.
- Host shaders are compiled with `glslangValidator --target-env vulkan1.3`.
- `third_party/fsr-vulkan` is built and linked.
- ImGui uses `imgui_impl_vulkan.cpp`.
- The shadPS4 `video_core` and shader-recompiler sources are globbed into `bbgpu`.
- Windows builds still link `Vulkan::Vulkan`.
- The Windows `bb-probe` target includes `src/vulkan_smoke.c`.

Consequently, replacing only device creation or adding a D3D12 device probe would not make the renderer portable.

## Architecture decision

Use a backend boundary, not a Vulkan-to-D3D12 wrapper written from scratch as the first step.

Proposed conceptual layout:

- `gpu/backend/` — backend-neutral interfaces for device, queue, resource, pipeline, synchronization and presentation.
- `gpu/backend/vulkan/` — existing desktop implementation, retained.
- `gpu/backend/xbox_d3d12/` — Xbox-specific D3D12.x implementation, built only with the supported Xbox toolchain.
- `gpu/shaders/` — source shaders and explicit per-backend compilation rules.
- `docs/` — capability matrix, memory budget and test results.

These are proposed destinations, not directories that already exist.

Do not assume Mesa Gallium's `d3d12` driver is a drop-in Vulkan-to-D3D12 translator. It is a Gallium driver implemented over D3D12. Likewise, vkd3d's usual D3D12-on-Vulkan direction does not directly solve Vulkan-on-Xbox.

## Phased implementation

### Phase 0 — Preserve a known baseline

1. Record the current branch head and clean build instructions.
2. Keep the existing Vulkan build unchanged.
3. Add Xbox-specific code behind an explicit build option; never silently switch the desktop target.

Exit criterion: the existing desktop target still configures and builds as before.

### Phase 1 — Native Xbox graphics proof

Build a minimal Xbox Dev Mode app with the supported Xbox toolchain and API headers. It should:

1. Create the supported D3D12.x device/context.
2. Create a command queue and command allocator/list.
3. Create a swap chain and present a clear-color frame.
4. Allocate, write, use, and release one small buffer and texture.
5. Log HRESULTs, adapter/device details available to the app, and timing.
6. Repeatedly resize/recreate presentation resources and exit cleanly.

Do not infer that hardware D3D12 is unavailable solely because a generic UWP `D3D12CreateDevice` probe fails. Confirm behavior using the Xbox-supported API/toolchain.

Exit criterion: repeatable hardware-backed presentation on the actual Series S. A software adapter is not a passing result.

### Phase 2 — Backend inventory

Before porting renderer code, identify and classify all uses of:

- Vulkan instance/device/queue and extension loading
- images, buffers, memory allocation and mapping
- descriptors and descriptor sets
- graphics/compute pipelines and shader modules
- barriers, fences, semaphores and queue submission
- swap chain, render passes/framebuffers and presentation
- timestamps/queries and device capability checks
- ImGui Vulkan backend
- Vulkan-based FSR/upscaling components

For each use, record source path, owning subsystem, Xbox replacement, and whether it blocks compilation or runtime.

### Phase 3 — Resource and command model

Implement resource lifetime and synchronization first. The backend must avoid unnecessary CPU-side staging and duplicate full-resolution resources. Prefer explicit ownership, pooled allocations where appropriate, and measured transient-resource lifetimes.

Do not map VMA calls one-to-one to a new allocator without checking Xbox memory APIs and placement/alignment requirements.

### Phase 4 — Shaders and pipelines

The current host shader build emits Vulkan 1.3-oriented outputs. Define the target shader format and compilation path supported by the Xbox toolchain, then port one simple vertex/pixel pipeline before compute/post-processing shaders.

Track shader compatibility separately from guest PS4 shader translation/recompilation. A host API change does not automatically solve guest shader semantics.

### Phase 5 — Incremental renderer bring-up

Suggested order:

1. Clear and present.
2. Vertex/index buffers and a textured triangle.
3. Upload and sample a texture.
4. Depth/stencil and render-target transitions.
5. Guest GPU command translation and shader pipeline creation.
6. UI/overlay.
7. Post-processing and upscaling.
8. Performance and memory tuning.

Keep features disabled until their correctness tests pass.

## Memory budget and measurements

The approximately 5 GiB Dev Mode cap must be confirmed for the actual package and runtime configuration. Do not hard-code a presumed usable amount.

Capture at minimum:

- process/package memory usage over time
- peak usage during boot, asset loading, scene transitions and shader compilation
- guest RAM allocations versus host renderer allocations
- texture and render-target dimensions/formats/sample counts
- upload/readback staging sizes and lifetimes
- shader/pipeline cache growth
- device allocation failures and HRESULTs

Add a repeatable stress test that creates and releases resources in batches. Record the failure threshold, recovery behavior, and memory after release. If the platform's accounting does not expose a separate GPU-memory figure, label the metric accordingly instead of treating process memory as GPU VRAM.

## Build-system direction

The current CMake file should eventually select dependencies by backend, for example:

- Vulkan target: Vulkan headers/loader, VMA, Vulkan ImGui backend, FSR Vulkan.
- Xbox target: Xbox SDK/GDKX D3D12.x headers/libraries and Xbox-compatible UI/upscaling implementations.

Do not add guessed library names or pretend the standard Windows SDK is equivalent to GDKX. Gate the Xbox target on the actual installed toolchain and document the exact supported SDK version.

## Immediate next engineering task

Perform a source inventory of `shadps4/video_core` and the local `shim` layer, starting from device creation, command submission, resource allocation, and presentation. Produce a path-by-path dependency table before making broad renderer edits.

## Acceptance criteria

- Desktop Vulkan target remains buildable.
- Xbox target uses the supported Xbox graphics API, not a software adapter.
- A clear-color frame is presented repeatedly on the Series S.
- Resource creation/release stress test is repeatable.
- Measured package-memory headroom is documented.
- No game dumps, keys, or copyrighted assets are added to the repository.
