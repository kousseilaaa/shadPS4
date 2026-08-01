# GoW3 Remastered performance investigation (fix/gow3)

Working notes from a performance investigation session (2026-08-01/02). Written to be reusable by
humans and AI agents continuing this work. All file:line references are as of commit `2535bc96`;
line numbers drift, but function names are stable anchors.

## Problem statement

God of War 3 Remastered on shadPS4: steady sub-20 FPS in specific game areas, recovering elsewhere,
while **CPU and GPU utilization both sit at 30–40%**. Low-FPS-with-low-utilization is the signature
of serialization/stalls, not compute limits. Not stutter/hitching (that would point at shader
compilation instead). Host: mid-range Windows PC. Config: defaults except
`readback_linear_images_enabled: true`, which this title needs to avoid texture corruption.

## Root cause (verified in source, fixed in `59bc98a6`)

With readback-linear-images on, every linear (or width ≤ 8) image touched as an RT/storage image is
queued for CPU writeback in `TextureCache::FindRenderTarget`/`FindTexture`
(`src/video_core/texture_cache/texture_cache.cpp`, search `readback_linear_images`). Additionally,
this branch's auto-exposure fix (`b54ed280`) queues the 1×1 R16G16Sfloat exposure RT **every frame**
(`vk_rasterizer.cpp`, search `AddDownload`).

`TextureCache::ProcessDownloadImages()` runs on the single GPU command-processor (CP) thread at
every `EventWriteEop`/`EventWriteEos`/`ReleaseMem` packet and at every submit
(`liverpool.cpp`, `vk_rasterizer.cpp:OnSubmit`). Before the fix it called
`DownloadImageMemory(id, sync=true)` whose sync branch is **`scheduler.Finish()` — a vkQueueSubmit
plus a full blocking wait for GPU idle — once per queued image**. Cost scales with how many linear
images an area's render path touches → steady, area-dependent FPS drops with idle hardware.

The frame pacer amplifies every stall: `VideoOutDriver::PresentThread`
(`src/core/libraries/videoout/driver.cpp`) consumes **at most one flip per vblank tick** and flip
events are only signalled from that thread's tick (never at GPU completion), so a frame missing its
slot waits a whole vblank period → FPS quantizes to 60/30/20/15.

### The fix

`ProcessDownloadImages` now uses the pre-existing deferred path: `DownloadImageMemory(id, false)`
records the copy and defers the guest-memory write via `Scheduler::DeferPriorityOperation`;
`PriorityPendingOpsThread` waits the GPU tick off the CP thread. All downloads in a batch share one
tick, and a single `scheduler.Flush()` replaces N submit+drain cycles. Guest memory receives the
data ~one GPU-batch later, which auto-exposure tolerates invisibly.

Runtime A/B + safety switch: new GPU setting **`readback_linear_images_sync`** (default `false`).
`true` restores byte-exact old behavior (data in guest memory before the EOP fence fires) without a
rebuild — use it if texture corruption ever reappears, or for baseline captures.

Correctness note for future work: the old sync path guaranteed writeback-before-EOP-fence. If some
other game/effect turns out to CPU-read a downloaded image immediately after EOP, async may be too
late for it — that's what the toggle (and a possible middle mode: one `Finish()` per batch) is for.

## Everything else found (ranked leads if the fix isn't enough)

1. **CPU↔GPU submission serialization**: `sceGnmSubmitDone` latches `submission_lock` when the CP
   thread isn't idle; the next `sceGnmSubmitCommandBuffers`/`sceGnmDingDong` blocks in
   `WaitGpuIdle()` (`gnmdriver.cpp`) until the CP thread drains everything and signals
   `InterruptId::GpuIdle` (`liverpool.cpp`, end of `Process` loop). Frame N+1 command building is
   strictly serialized behind frame N parsing. Already instrumented (`HLE_TRACE` on `WaitGpuIdle`).
2. **Flip pacing quantization**: one flip per vblank tick, `flip_rate` divides further
   (`flip_rate=1` → 30 fps ceiling at 60 Hz). `AccurateTimer` carries wait-debt and refuses to draw
   early. Fifo present mode would add a second driver-level vsync on top (default is Mailbox).
   A stall of even 1 ms past the tick costs 16.6 ms.
3. **Windows timing** (fixed in `2535bc96`): no `timeBeginPeriod` existed anywhere; `AccurateSleep`
   created a default-resolution waitable timer per call → sleeps overshooting by up to ~15.6 ms.
   Now: per-thread `CREATE_WAITABLE_TIMER_HIGH_RESOLUTION` timer + `timeBeginPeriod(1)` at startup.
4. **Buffer readback stall-the-world** (inactive at current config): any guest CPU read of
   GPU-written memory faults → `BufferCache::ReadMemory` → blocking `SendCommand<true>` round-trip
   to the CP thread → `DownloadBufferMemory` → `scheduler.Finish()`. Gated by `readbacks_mode`
   (Disabled by default — keep it Disabled for this game).
5. **Shader compilation is fully synchronous on the CP thread** (`vk_pipeline_cache.cpp`,
   `CompileModule`) — first-encounter hitches. Mitigation: `pipeline_cache_enabled: true` (disk
   cache + eager `WarmUp()` preload at boot; default is OFF).
6. **Synchronous logging by default**: `Log.sync` defaults to `true` (spdlog direct file writes on
   hot threads). Set `"sync": false`. Log macro args are always evaluated; `LOG_DEBUG` is not
   compiled out in release; each call does a hash-map lookup + thread-name `std::string` copy.
7. **Per-draw costs worth knowing**: `Scheduler::PopPendingOperations()` +
   `MasterSemaphore::Refresh()` (a driver call) run at the top of every draw/dispatch;
   `uses_dma` shaders re-synchronize **all** mapped buffer ranges per draw
   (`Rasterizer::BindResources`); GpuModified texture refresh hashes the full image per mip with
   XXH3 (`RefreshImage`); partial buffer uploads still emit whole-buffer barriers;
   `CopyFromLastRt` took a global mutex + map lookup per texture bind (now has an unlocked
   empty-map fast path, valid because only the CP thread mutates `last_rt_address_`).
8. **Threading trivia**: all GPU rings (1 GFX + 56 compute) run as coroutines on ONE CP thread,
   round-robin, waits are `while(!cond) co_yield` spins. Guest thread affinity is a deliberate
   no-op (`Pthread::SetAffinity` body commented out); guest priorities aren't forwarded to the
   host. Equeue "small timer" waits busy-spin with `yield` under 1.2 ms
   (`HrTimerSpinlockThresholdNs`).
9. **Config is JSON in this fork** (`<user>/config.json`, per-game
   `custom_configs/<SERIAL>.json`; definitions in `src/core/emulator_settings.h`). `config.toml`
   is only a one-time migration source. The loader merges file keys over serialized defaults, so
   adding new settings fields is backward compatible.

## Profiling infrastructure (how to measure)

- **Tracy** is integrated; `TRACY_ENABLE` is auto-ON for every build type except Release
  (`externals/CMakeLists.txt`). `TRACY_ON_DEMAND=ON` (zero cost until GUI connects),
  `TRACY_ONLY_LOCALHOST=ON`. The GUI must be **v0.11.1** (shadps4-emu fork tag) or it refuses the
  protocol. Macro layer in `src/common/debug.h`: `RENDERER_TRACE`/`HLE_TRACE` (zones), `TRACE_HINT`,
  `FRAME_END` (FrameMark, fires only on a real flip in PresentThread). `TRACY_GPU_ENABLED` is
  hardcoded `0` in debug.h — flip to `1` + rebuild for GPU-side (Vulkan timestamp) zones.
  Fiber markers for the CP coroutines exist but `TRACY_FIBERS` is off (upstream: instability).
- **Zones/plots added this session** (`68fc4036`): `ProcessDownloadImages` (+ count),
  `DownloadImageMemory` (+ `DownloadBytes` plot), `Scheduler::Finish` (the smoking-gun zone —
  its width = GPU drain time), `Scheduler::SubmitExecution`, blocking fallback of
  `MasterSemaphore::Wait`, `Rasterizer::OnSubmit`, `FlipQueueDepth` plot + flip zone in
  the videoout driver.
- **In-game tools**: F10 = FPS overlay; Ctrl+F10 = advanced menu (frame graph, frame/presenter
  times, flip + Gnm submit counts); Ctrl+Alt+F9 = PM4 frame dump viewer; F12 = RenderDoc capture
  (needs `renderdoc_enabled`); `null_gpu: true` isolates CPU vs GPU bottlenecks.

## Measurement workflow (Windows)

1. Build the `x64-Clang-RelWithDebInfo` preset (submodules initialized, incl. `externals/tracy`).
2. Config: `readback_linear_images_enabled: true`, Log `"sync": false`,
   `"show_fps_counter": true`. Optionally `pipeline_cache_enabled: true` (Vulkan section).
3. Baseline: set `readback_linear_images_sync: true`, walk to a slow area, connect Tracy GUI,
   capture ~30 s. Fix run: toggle to `false`, same spot, capture again.
4. Read: on thread `shadPS4:GpuCommandProcessor`, per-frame width/count of `ProcessDownloadImages`
   and `Scheduler::Finish` zones (Statistics view); plots `PendingImageDownloads`, `DownloadBytes`;
   on `shadPS4:PresentThread`, FrameMark spacing and `FlipQueueDepth` (0 = starved, >0 = paced).
5. **Watch item**: if `DownloadBytes` totals approach ~16 MB/frame, bump `DownloadBufferSize`
   (32 MB) in `src/video_core/buffer_cache/buffer_cache.cpp` — otherwise the download staging
   ring's wraparound wait silently reintroduces a CP-thread stall.

## Changes shipped on this branch (this session)

| Commit | What |
|---|---|
| `59bc98a6` | Async + batched linear-image readbacks; `readback_linear_images_sync` toggle; `AddDownload` locking; `CopyFromLastRt` empty fast path |
| `68fc4036` | Tracy zones/plots on all suspected stall paths |
| `2535bc96` | Windows: high-resolution waitable timer in `AccurateSleep` (per-thread, pre-1803 fallback) + `timeBeginPeriod(1)` at startup, link `winmm` |

Earlier branch commits (previous sessions): `b54ed280` auto-exposure chain fix (source of the
per-frame 1×1 download), `5f726882`/`70170769` effective depth-stencil state refinements.
