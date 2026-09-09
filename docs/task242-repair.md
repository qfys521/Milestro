# Task #242 repair and local verification - 2026-09-09

Base: `5054c92d03cb43eeb428f0257738b1df74a78d2e` on
`fix/android-platform-readiness`. This report describes the working-tree repair
on that base, not an independent review or a release approval.

## Dispatcher P2

An unregistered Direct submit ticket has no ownership of the shared route queue.
After the bounded retired history turns over, the dispatcher cannot distinguish
an old ticket from an early ticket. The old implementation failed every queued
Direct submission in that case.

`SubmitDirectTicket` now records only the bounded early marker. It does not drain
or complete queued work. Existing registered-ticket and recently-retired-ticket
handling is unchanged. Only the matching prepare consumes an early marker and
fails the batch it drains. Thus submit-before-prepare still fails closed, but
failure publication is deferred until that prepare establishes batch ownership.
An unrelated prepare is unaffected by the marker. Both histories remain bounded
at 256 entries; no payload ABI or managed/native ticket encoding changes here.

Regression coverage in `test_unity_render_vulkan_production.cpp`:

- `SubmitBeforePrepareLeavesQueuePendingUntilMatchingPrepare`: the early event
  leaves work pending; matching prepare fails it; duplicates do not submit it.
- `EvictedRetiredSubmitTicketsLeaveFreshQueuedWorkPending`: retire 257 empty
  batches through real prepare/submit dispatcher callbacks, then enqueue fresh
  work through `MilestroUnityRenderEnqueueSubmission`. Check recent retired,
  evicted retired, duplicate, and unknown tickets, including another 257 events
  turning over the early history. Fresh work stays pending, prepares normally,
  and is drawn/submitted exactly once. No extracted dispatcher/history helper is
  used; the suite still uses its existing fake Unity callbacks and Ganesh Mock
  context, not a real GPU.

## Build blockers discovered while running the suite

The available VS 2022 MSVC compiler was found using the Visual Studio CMake
generator (the previous Ninja compiler discovery failure was not proof that no
host compiler was installed). The initial full build reached Skia successfully,
but failed in ICU code generation/flags before the production tests could link:

- Generated data used GCC `__attribute__((aligned(16)))` and an empty inferred
  array. Emit standard `alignas(16)` inside an `extern "C"` block and a one-word
  zero placeholder when data embedding is disabled. The C symbol and alignment
  are retained; actual embedded words are unchanged when embedding is enabled.
- ICU passed `-Wno-deprecated-declarations` to MSVC, which rejects it with D8021.
  Use `/wd4996` for MSVC, retaining the existing option for other toolchains.

`python -B tests/icu/test_make_data_cpp.py` tests the placeholder and complete
embedded output separately. These portability changes are separate from the
Direct queue ownership fix.

## Test fixture corrections

The expanded Vulkan-found CTest run exposed two existing test setup errors:
`RejectionNeverOverwritesLastAcceptedSubmission` accepted an unregistered Vulkan
target, and the staging queue test used invented target handles. Production
Vulkan enqueue correctly rejects those handles when compiled in.

The backend-neutral diagnostics acceptance now uses OpenGLES; its stale-epoch,
backend-mismatch, and unknown-backend rejection checks remain. Staging cumulative
queue coverage was moved to `VulkanDispatcherProductionTest`, which loads fake
Unity interfaces and registers a target via the public C export. It preserves
all prior pending/superseded assertions and additionally dispatches the surviving
clear/cumulative submissions to Drawn before target destruction. No production
validation was relaxed or test skipped.

## Local host setup

Windows x64, VS 2022 Community / MSVC 19.44.35228.0, CMake 4.3.3;
Python 3.12.10 runs the codegen tests. Android builds use the existing Unity
6000.3.7f1 NDK 27.2.12479018 build trees (ARM64 API 26 Vulkan / API 23 GLES).

The first Windows suite run threw access violations with the system's
`msvcp140.dll` / `vcruntime140.dll` version 14.00.24215.1. Running the same binary
with the VS-shipped 14.44.35112 CRT DLLs copied **only into the build's bin
directory** passed all 20 tests. No system runtime files were replaced. Retain
this app-local runtime setup when reproducing the test commands below.

## Executed results

All results below are local implementation evidence, not independent signoff.
The final selected desktop CTest set is four entries: ICU (4 cases), ICU codegen
(2 cases), UnityRenderPayload (28 cases), and VulkanProduction (21 cases).

| Check | Result | Local log under `build/` |
| --- | --- | --- |
| Original dispatcher + new eviction regression | Expected FAIL: fresh submission is Failed (-1), not Pending (0) | `task242-red-regression.log` |
| Repaired dispatcher, before fixture corrections | PASS, 20/20 production cases | `task242-desktop-fallback-tests.log` |
| Final Vulkan-found static build/link and selected CTest | PASS, 4/4 CTest entries (55 cases) | `task242-desktop-found-final-build.log`, `task242-desktop-found-final-tests.log` |
| Final Vulkan-disabled/fallback static build/link and selected CTest | PASS, 4/4 CTest entries (55 cases) | `task242-desktop-fallback-final-build.log`, `task242-desktop-fallback-final-tests.log` |
| Platform contracts | PASS, 24/24 | `task242-contract-tests.log` |
| Android ARM64 API 26 Vulkan native library | PASS | `task242-android-vulkan.log` |
| Android ARM64 API 23 GLES native library | PASS | `task242-android-gles.log` |
| Vulkan-found Release static archive test-hook audit | PASS: DirectBackend present, no DirectAdapterTest definitions | `task242-desktop-symbol-audit.log` |
| Android Vulkan library/object/export test-hook audit | PASS: no DirectAdapterTest definitions/exports | `task242-android-symbol-audit.log` |

The RED/GREEN comparison temporarily restored **only**
`MilestroUnityRenderDispatcher.cpp` from the base commit, keeping the regression
and ICU build fixes. The same configured production executable was rebuilt and
failed the new regression; restoring the repaired dispatcher and rebuilding
passed the entire then-current 20-case suite. This isolates the P2 behavior
without claiming the unmodified whole base compiled on MSVC.

### Configuration and reproduction

The same desktop build directory was reconfigured between the two modes; its
final on-disk state is Vulkan-disabled/fallback. The Vulkan-found archive audit
was performed before that switch, with its results preserved in the named logs.

The Vulkan-found build used bundled Khronos Vulkan headers and an x64 import
library generated with MSVC dumpbin/lib from the 265 `vk*` exports of the real
system loader `C:\Windows\System32\vulkan-1.dll` (1.4.341.0), not a stub library.
Inputs are `build/task242-vulkan-loader.def` and
`build/task242-vulkan-loader.lib`. This satisfies CMake's Vulkan discovery/link
path; tests still use fake Unity/Vulkan callbacks and Ganesh Mock, not a driver
rendering workload. No full Vulkan SDK installation or validation layer is
implied.

Run from the repository root, retaining the app-local CRT setup described above:

```powershell
cmake -S . -B build/desktop-vulkan-review-vs -G "Visual Studio 17 2022" -A x64 `
  -DMILESTRO_ENABLE_CLI=OFF -DMILESTRO_ENABLE_TESTS=ON `
  -DMILESTRO_ENABLE_DESKTOP_VULKAN_RENDER=ON -DMILESTRO_BUILD_SHARED_LIBS=OFF `
  -DCMAKE_DISABLE_FIND_PACKAGE_Vulkan=FALSE `
  -DVulkan_INCLUDE_DIR=E:/Code/Milestro/ext/skia/third_party/externals/vulkan-headers/include `
  -DVulkan_LIBRARY=E:/Code/Milestro/build/task242-vulkan-loader.lib
cmake --build build/desktop-vulkan-review-vs --config Release `
  --target MilestroTest_UnityRenderVulkanProduction MilestroTest_UnityRenderPayload MilestroTest_Icu --parallel 6
ctest --test-dir build/desktop-vulkan-review-vs -C Release `
  -R '^MilestroTest_(UnityRenderVulkanProduction|UnityRenderPayload|Icu|IcuCodegen)$' --output-on-failure -V
# Exercise the fallback source/macro path, then repeat the same build and CTest:
cmake -S . -B build/desktop-vulkan-review-vs -DCMAKE_DISABLE_FIND_PACKAGE_Vulkan=TRUE

cmake --build build/platform-contracts --config Release
ctest --test-dir build/platform-contracts -C Release --output-on-failure
python -B tests/icu/test_make_data_cpp.py
cmake --build build/android-unity-27.2.12479018-arm64-v8a-api26-vulkan --parallel 4
cmake --build build/android-unity-27.2.12479018-arm64-v8a-api23-gles --parallel 4
```

Vulkan-found project definitions were inspected: the production `Milestro`
target has Vulkan enabled without `MILESTRO_UNITY_RENDER_VULKAN_PRODUCTION_TEST`;
the production-test executable compiles DirectAdapter with that test macro.
The static archive audit also requires a real `DirectBackend` definition so
absence of hooks is not inferred from an empty/non-Vulkan library.

## Scope limits

Only the selected native suites above were run, not every repository test.
No ARMv7 full build or new managed compilation was run.
No claim is made about real Vulkan execution, validation layers, Unity APKs,
ADB/device behavior, pause/resume/resize/recreate, sanitizers, or the historical
Direct device-destruction/old-driver risk. The POSIX protected-memory death test
is not compiled on Windows. Managed/native must still be delivered together for
the earlier numeric-ticket protocol change; this repair does not change it again.
No external review workflow, paused task, merge, or release is resumed by this
local implementation work.
