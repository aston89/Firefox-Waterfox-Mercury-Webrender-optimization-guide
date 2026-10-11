# Firefox / Waterfox / Mercury: WebRender and JavaScript Optimization Guide

This guide collects **workload-specific experiments** for Firefox-family browsers, especially systems where GPU compositing has shown excessive overhead and software WebRender performs better.

It is not a universal “maximum performance” preset. Firefox, Waterfox and Mercury can differ by version, build flags, graphics backend and SpiderMonkey implementation. A preference that helps one workload may increase memory use, compilation cost, CPU contention or input latency in another.

> [!IMPORTANT]
> **Start from the browser's defaults and benchmark one change at a time.** Keep a backup of the profile, record the exact browser/build and use `about:support` to verify the active graphics backend. Reset an experimental preference with its reset button rather than copying a numeric “default” from another Firefox release.

## Field note: very large, modern single-page applications

The aggressive JavaScript/JIT settings in §4 helped with hangs and responsiveness in an earlier workload on the author's system. After a major update to the ChatGPT single-page application in October 2026, the same general tuning approach was no longer a good fit: the page became much heavier, memory spikes appeared with aggressive JIT environment overrides, and scrolling remained sluggish.

The environment overrides were removed and the custom JIT preferences were reset. This is an **observed compatibility regression on one system**, not proof that the settings are harmful in every browser or that JIT tuning alone caused every slowdown. The practical recommendation is to test the current application with the browser's normal JIT and garbage-collection defaults first. Keep the old settings only as a clearly labelled, optional legacy experiment.

---

## 1. GPU acceleration and layer compositing: workload matters

GPU acceleration often improves browser rendering, but it is not automatically the fastest path on every machine. Depending on the GPU, driver, browser build, display configuration and page workload, GPU compositing can introduce costs such as:

- GPU texture allocation and transfers;
- CPU/GPU synchronization;
- additional graphics-process memory;
- compositor bookkeeping and driver overhead.

These costs can be especially visible on some systems with complex, frequently changing pages. On the author's setup, software rendering has been preferable to the tested GPU-accelerated configuration, including during work with large ChatGPT conversations.

That result should **not** be generalized to all hardware or websites. Compare both paths on the actual machine and workload before deciding which to keep.

## 2. Software WebRender

Firefox-family builds can expose a software-rendering path for WebRender. The precise rasterization, presentation and compositing path depends on the browser version, operating system and graphics backend. Do not assume that enabling software WebRender means the GPU is still used only to present the final frame, or that every form of GPU texture/compositor overhead disappears.

To test software WebRender on a supported build:

1. Open **Settings → General → Performance** and deselect **Use recommended performance settings** if needed to expose the performance controls.
2. Open `about:config`.
3. Search for `gfx.webrender.software` and set it to `true` if the preference exists and the build honors it.
4. Restart the browser and inspect `about:support` → **Graphics** to confirm the active backend and whether the intended path is actually in use.

Keep other graphics preferences unchanged during this test. In particular, do not combine the test with changes to `layers.acceleration.force-enabled` or `layers.gpu-process.enabled`: change one variable at a time so the result is interpretable.

---

## 3. WebRender tuning

The preferences below concern different parts of the rendering pipeline. Their availability and effect can vary between releases and forks. Treat the values as **test candidates**, not defaults that every user should apply.

### 3a. Update rectangles and surface pool

Picture tiles and update rectangles are related, but they describe different things:

- **Picture tiles** partition rendered picture content into regions that can be cached and invalidated.
- **Update rectangles** represent changed regions passed through later partial-update/presentation stages.
- **Surface pools** help manage intermediate rendering surfaces and their reuse.

Simplified pipeline:

```text
picture content
    ↓
picture tiles / retained rendering
    ↓
changed regions
    ↓
update rectangles
    ↓
partial presentation / compositor
    ↓
presentation
```

These mechanisms are not interchangeable. A `512×512` tile size does not imply any particular number of update rectangles. Smaller regions can reduce unnecessary work for small changes, but they also add bookkeeping; larger regions reduce management overhead but may cause more content to be rerasterized.

The following values are one **moderate experimental profile** used on the author's software-rendering setup:

```text
gfx.webrender.compositor.max_update_rects = 16
gfx.webrender.max-partial-present-rects = 16
gfx.webrender.compositor.surface-pool-size = 64
```

These are not universal sweet spots. Too-small limits or pools may constrain reuse or cause less efficient updates; unnecessarily large values may increase memory and bookkeeping without improving frame delivery. Do not assume that increasing a limit increases performance.

### 3b. Picture tile dimensions

```text
gfx.webrender.picture-tile-width
gfx.webrender.picture-tile-height
```

The exact preferences exposed by a build may vary. Values of `256` and `512` pixels have both been useful as experiment points, but no single tile dimension is optimal for every display resolution or page. Start with the build's own values and compare one change at a time while scrolling the same content.

- Smaller tiles can make invalidation more local, but increase the number of regions to manage.
- Larger tiles can reduce bookkeeping, but may rasterize more unchanged content.
- The best result depends on update patterns, scene complexity, memory pressure and the active rendering backend.

Do **not** mathematically match tile dimensions to update-rectangle limits; they govern different stages.

### 3c. Blob tiles

Blob-image caching may be used for text and vector content. Its precise behavior and available preferences depend on the build.

```text
gfx.webrender.blob-tile-size = 256
```

Use this only if the preference exists and the browser build actually honors it. Smaller blob tiles may improve reuse when small portions change, while increasing tile-management overhead; larger tiles may reduce bookkeeping but rerasterize more content after invalidation. Keep the build's default unless testing shows a repeatable improvement.

### 3d. Text rendering and font cache

Text rendering can involve cached vector/blob content and a separate glyph/font cache. These caches work at different levels, and changing one does not guarantee a proportional improvement in scrolling.

An experimental value used on the author's system is:

```text
gfx.content.skia-font-cache-size = 64
```

The cache size's meaning and limits are build-specific. Test a small range only when text rendering is demonstrably the bottleneck; increasing a cache can trade memory for reuse and is not guaranteed to improve performance.

### 3e. ClearType and text sharpness

The following preferences have been used to test Windows text-rendering behavior:

```text
gfx.font_rendering.cleartype_params.rendering_mode = 0
gfx.font_rendering.cleartype_params.cleartype_level = 0
```

These values affect text appearance and rendering behavior, but do not assume that they produce a measurable performance gain on every build. Disabling or changing ClearType can make text less sharp, particularly at small sizes or on high-DPI displays. Keep the setting that provides acceptable readability unless a controlled test demonstrates a worthwhile benefit.

### 3f. Retained display lists for stacking contexts

```text
layout.display-list.retain.sc = true
```

This preference concerns retained display-list behavior for stacking contexts in builds that support it. Retaining data can avoid rebuilding some information, but benefit depends on how the page invalidates and changes. Treat it as a build-specific experiment and test scrolling, layout changes and memory use; reset it if the result is neutral or worse.

### 3g. Thread-local arenas for WebRender threads

Some builds may expose preferences such as:

```text
gfx.webrender.frame-builder-thread-local-arena
gfx.webrender.scene-builder-thread-local-arena
```

These relate to allocator behavior for particular WebRender threads, not to the rendering pipeline itself. A dedicated thread-local arena can reduce some allocation-lock contention, but it can also change memory behavior. Availability and implementation are build-dependent. Do not create missing preferences unless the relevant build documents them, and do not enable them without a repeatable benchmark.

### 3h. JavaScript Web Worker count is not the WebRender worker pool

`dom.workers.maxPerDomain` is a ceiling on JavaScript Web Workers for a domain; it is **not** the number of threads Firefox creates and does not configure WebRender's internal worker pool.

```text
dom.workers.maxPerDomain = 512
```

On builds where `512` is the default, leaving it there generally means allowing a high ceiling, not creating 512 threads. Actual parallelism depends on the workers a page creates and how it distributes its work. The page's main-thread JavaScript, rendering, garbage collection and compositing are distinct parts of the system; increasing a worker-count ceiling cannot automatically parallelize main-thread work.

### 3i. WebRender batching lookback

```text
gfx.webrender.batching.lookback = 40
```

The batching lookback controls how far the batching logic searches for compatible rendering primitives. A larger value may find more batching opportunities while requiring more CPU-side search/bookkeeping. This is a heuristic, not a frame queue or latency setting. The default varies by version; retain the default unless profiling on the target build shows a repeatable gain.

### 3j. Precise gradient calculations

Some builds expose preferences such as:

```text
gfx.webrender.precise-conic-gradients = false
gfx.webrender.precise-conic-gradients-swgl = false
gfx.webrender.precise-linear-gradients-swgl = false
gfx.webrender.precise-radial-gradients = false
gfx.webrender.precise-radial-gradients-swgl = false
```

These are build-dependent and may change rendering precision or appearance. Setting them to `false` is an experiment, not a guaranteed performance improvement. Confirm that each preference exists, compare visual output, and reset any change that does not show a measurable benefit on the target workload.

---

## 4. JavaScript / SpiderMonkey JIT and script loading

**This section contains aggressive, workload-sensitive experiments. It is not the recommended baseline for current large SPAs.** These settings previously helped with hangs and responsiveness in the author's older workload. In a newer and substantially heavier ChatGPT SPA, aggressive JIT environment overrides coincided with very large memory spikes and were removed. Custom JIT preferences were then reset to the browser defaults for a clean comparison.

The JIT changes below can alter warm-up, compilation volume, generated machine code, garbage-collection concurrency and startup work. A lower threshold does not automatically mean faster interaction: it can trigger more compilation and code generation for scripts that do not benefit from it, and large applications can keep substantial amounts of code and runtime data alive.

### Recommended starting point: restore the defaults

For a current, memory-heavy SPA, first reset the following preferences in `about:config` if you customized them through this guide:

```text
javascript.options.blinterp.threshold
javascript.options.baselinejit.threshold
javascript.options.ion.threshold
javascript.options.ion.frequent_bailout_threshold
javascript.options.inlining_bytecode_max_length
javascript.options.concurrent_multiprocess_gcs.cpu_divisor
javascript.options.concurrent_multiprocess_gcs.max
javascript.options.mem.gc_max_parallel_marking_threads

dom.script_loader.external_scripts.speculate_async.enabled
dom.script_loader.external_scripts.speculate_link_preload.enabled
dom.script_loader.delazification.strategy
dom.script_loader.delazification.max_size
dom.script_loader.delazification.min_mem
dom.script_loader.bytecode_cache.strategy
javascript.options.baselinejit.offthread_compilation_strategy
```

Not every preference exists in every release or fork. Reset only entries that appear in `about:config`; use the reset button so the browser selects the default for **that exact build**. Restart after changing preferences that are read during startup.

Also remove any `JIT_OPTION_*` variables from the environment or browser-launch batch file when returning to the baseline. SpiderMonkey reads these overrides during startup, so restart the browser completely after removing them.

### 4a. JIT warm-up thresholds - aggressive profile

Legacy test values:

```text
javascript.options.blinterp.threshold = 1
javascript.options.baselinejit.threshold = 10
javascript.options.ion.threshold = 300
```

These lower the warm-up thresholds so scripts can reach higher compilation tiers sooner. Common Firefox defaults in the source documentation are approximately `10`, `100` and `1500` respectively, but defaults must be checked against the target build.

Earlier tier-up can help hot code, but can also compile more code sooner, increase compilation activity and consume more memory. These values should be tested as a group only after establishing a default-profile baseline, then measured individually. Do not assume that an earlier tier-up improves responsiveness for every SPA.

### 4b. Ion frequent-bailout threshold - aggressive profile

Legacy test value:

```text
javascript.options.ion.frequent_bailout_threshold = 50
```

This preference affects how many frequent bailouts are tolerated before code is treated as having a recurring bailout problem. Raising it can change how long an optimizing-JIT script remains eligible to run, but does not make speculative optimization universally more efficient. Behavior and default values can vary by build; reset it unless testing identifies a repeatable benefit.

### 4c. Aggressive inlining - high memory-risk experiment

Legacy test value:

```text
javascript.options.inlining_bytecode_max_length = 4096
```

This changes the bytecode-size threshold used when considering functions for inlining. It does **not** force every eligible function to be inlined, but a much larger threshold can expand the set of inline candidates and contribute to larger generated code and longer compilation.

A setting of `4096` is intentionally far more aggressive than the typical default used by relevant Firefox builds. Combined with other aggressive JIT overrides, it should be treated as an experiment, not a routine optimization for large SPAs. Restore the default first if memory usage rises, startup slows, or scrolling becomes less responsive.

### 4d. Concurrent JavaScript garbage collection

Legacy test values:

```text
javascript.options.concurrent_multiprocess_gcs.cpu_divisor = 1
javascript.options.concurrent_multiprocess_gcs.max = 0
```

These settings affect the allowed concurrency of garbage collection across processes on builds that expose them. A smaller CPU divisor can permit more concurrent GC activity on high-core-count machines. More concurrency can help in some circumstances, but it can also increase CPU contention, memory pressure and simultaneous activity across processes. `max = 0` semantics should be checked for the target build rather than assumed from another version.

More cores do not automatically justify a divisor of `1`. Reset these preferences for the baseline, and only experiment with GC concurrency when profiling demonstrates that GC-not script compilation, layout or the compositor-is the bottleneck.

### 4e. Speculative off-main-thread script compilation

Legacy test values:

```text
dom.script_loader.external_scripts.speculate_async.enabled = true
dom.script_loader.external_scripts.speculate_link_preload.enabled = true
```

Where available, these preferences enable speculative work for particular external-script loading paths. Speculation can move some work earlier, but it can also do work for code that is delayed, unused or invalidated. The names, defaults and implementation are version-dependent. Reset to defaults when diagnosing memory spikes or regressions in a large application.

### 4f. Eager delazification for large script sets

Legacy test values:

```text
dom.script_loader.delazification.strategy = 255
dom.script_loader.delazification.max_size = -1
dom.script_loader.delazification.min_mem = 2
```

This profile requests more eager function parsing/delazification and removes the usual script-size cutoff where the build interprets `-1` that way. It may reduce later parsing delays for some applications, but it can also increase upfront CPU work and memory use, especially when the application ships a large script bundle.

This is a high-risk experiment for large SPAs, not a general speed setting. Restore the build's defaults unless cold-load measurements show a repeatable net benefit without worse memory or interaction latency.

### 4g. Bytecode cache strategy

Legacy test value:

```text
dom.script_loader.bytecode_cache.strategy = -1
```

This is a script-loader implementation detail whose accepted values and behavior are version-dependent. Do not assume that a negative value universally means “cache more” or that it improves repeat loads on every fork. The default in some Firefox versions is `0`; verify the exact implementation before changing this preference. Keep the default while diagnosing a current SPA regression.

### 4h. SpiderMonkey `JIT_OPTION_*` environment overrides - legacy only

The following environment-variable profile is **not recommended as a default configuration**. It is retained here for reproducibility and testing on builds that support these overrides.

Example Windows batch fragment from the earlier experimental profile:

```bat
set "JIT_OPTION_trialInliningWarmUpThreshold=50"
set "JIT_OPTION_trialInliningInitialWarmUpCount=30"
set "JIT_OPTION_inliningEntryThreshold=30"
set "JIT_OPTION_smallFunctionMaxBytecodeLength=2000"
set "JIT_OPTION_ionMaxScriptSize=5000000"
set "JIT_OPTION_ionMaxLocalsAndArgs=50000"
set "JIT_OPTION_branchPruningThreshold=2000"
START "" "%cd%\browser.exe" --profile "%~dp0USER_DATA"
```

SpiderMonkey builds that support them read `JIT_OPTION_<name>` overrides during startup. The variable names and defaults are implementation details and may differ by version or fork; confirm support in the source for the exact binary. A complete browser restart is required after changing these variables.

The values above make several compilation/inlining limits more permissive than the usual defaults. They can produce substantially more compiler work and generated code. In the author's testing, aggressive JIT environment overrides that had helped an older workload were associated with very large memory spikes on the newer ChatGPT SPA; removing the overrides was necessary to return to a stable baseline. This is an observed workload-specific result, not a universal RAM figure or guaranteed behavior.

An older note in this guide warned that setting `JIT_OPTION_inliningEntryThreshold` to `10` could produce transient memory spikes approaching 16 GB in a large SPA. Treat that figure as an anecdotal observation from one test environment, not a prediction. More paging capacity is not a substitute for controlling memory pressure.

### 4i. Experimental parallel GC marking

Some Firefox-family builds may expose:

```text
javascript.options.mem.gc_max_parallel_marking_threads = 8
```

This controls a maximum number of parallel marking threads where supported. The value is a ceiling, not a promise that eight threads are always used or that GC becomes eight times faster. Keep the preference at its build default unless the relevant version and workload have been profiled. On large SPAs, test peak memory and interaction latency as well as total GC time.

### 4j. Baseline JIT off-thread compilation strategy

Some newer or fork-specific builds may expose:

```text
javascript.options.baselinejit.offthread_compilation_strategy = 3
```

The mapping of numeric values to compilation strategies is version-specific. Do not assume that `3` always means the same thing across Firefox, Waterfox and Mercury builds. If the preference exists, consult source or release-specific documentation before changing it, then compare cold-load and warm-load behavior. A faster initial load does not necessarily imply smoother interaction after the entire SPA has initialized.

---

## 5. Testing methodology

For reproducible tuning, record the exact browser name/version, operating system, graphics driver, rendering backend and application version. Use the same long conversation or representative page for each run.

1. Establish a baseline with default JIT and script-loader preferences.
2. Restart the browser between changes that are read at startup.
3. Test only one preference or tightly related group at a time.
4. Measure at least cold load, warm load, idle memory, peak memory, scroll responsiveness and recovery after a long session.
5. Repeat each test; a single run can be affected by caching, background tasks and network variability.
6. Keep a change only when the gain is repeatable and does not introduce a worse memory or responsiveness regression.

If scrolling remains sluggish after resetting JIT settings, disable extensions one at a time and compare the same page. An extension that changes conversation history or DOM structure may reduce the rendered node count yet introduce work during scrolling or message loading. Do not attribute every scroll hitch to WebRender or the JIT without profiling.

---

## 6. Further reading

- [Software rendering and GPU acceleration notes](https://github.com/aston89/Firefox-Waterfox-Mercury-Webrender-optimization-guide/blob/main/DOM_SINGLE_THREADED.md)
- [Browser rendering pipeline](https://github.com/aston89/Firefox-Waterfox-Mercury-Webrender-optimization-guide/blob/main/RENDERING_PIPELINE.md)
- [GPU layers](https://github.com/aston89/Firefox-Waterfox-Mercury-Webrender-optimization-guide/blob/main/GPU_LAYERS.md)
- [JavaScript compilation notes](https://github.com/aston89/Firefox-Waterfox-Mercury-Webrender-optimization-guide/blob/main/JS_COMPILE.md)
- [Mozilla Firefox Source Docs: How SpiderMonkey optimizes JavaScript](https://firefox-source-docs.mozilla.org/js/how-we-optimize.html)
- [Mozilla Searchfox: Firefox source and generated preference definitions](https://searchfox.org/firefox-main/source/)

---

## Summary

- Software WebRender can be the better path on a particular system, but GPU-versus-CPU rendering is hardware- and workload-dependent.
- WebRender tile, update-rectangle, cache and batching preferences are experiments; do not assume larger values are faster.
- The aggressive JIT and script-loader profile is now labelled **legacy/experimental** because its benefits did not carry over to the newer, much heavier ChatGPT SPA in the author's testing.
- For current large SPAs, start with the browser defaults, then profile before reintroducing one tuning change at a time.
