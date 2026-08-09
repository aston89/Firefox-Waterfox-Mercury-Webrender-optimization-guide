## Firefox Waterfox Mercury : Webrender optimization guide
This guide explains a highly efficient configuration for WebRender in software mode. The goal is maximum performance and minimal CPU/RAM usage, without relying on GPU-accelerated layer compositing, which often generates massive overhead.

---

## 1. The Problem with GPU acceleration and layer compositing
Webrender based browsers attempt to accelerate rendering using GPU and in theory this should improve performance but it often causes:
* Massive **texture allocation**
* Constant **CPU ↔ GPU synchronization**.
* Increased **memory usage**
* Significant **bookkeeping overhead**

Each page layer becomes a separate GPU texture. The browser must constantly:
* Upload textures
* Synchronize layers
* Track updates
* Composite the final frame

For the vast majority of real-world websites this introduces **more overhead than benefits** even for complex pages such as:
* YouTube
* Google Maps
* Large ChatGPT conversations
* Social media feeds

Those mentioned above often perform **better without GPU layering**.

---

## 2. Software WebRender
WebRender can operate in **pure software raster mode**, while still using **D3D11 only as the presenter**.
In this configuration:
* The **CPU performs rasterization**
* The **GPU only presents the final frame**
* No texture layering overhead occurs

Enabling software WebRender:
* Tool / settings / Search "performance" and deselect "Use recommended performance settings"
* type in the url'bar "about:config" and press enter
* search for "gfx.webrender.software" and double click it, the value "true" shold appear right to it.

---

## 3. The Critical Optimizations
These settings operate at different levels of the WebRender rendering and presentation pipeline.
**Picture tiles** determine how rendered picture content is spatially partitioned into independently managed regions. Changing the tile dimensions therefore changes the granularity at which WebRender can cache, invalidate and rebuild rendered content.
**Update-rect limits**, on the other hand, operate later in the pipeline. They limit how many separate changed regions can be propagated as partial updates toward the compositor/presentation stage. They do not define the tile geometry itself; the resulting update rectangles may cover one or several tiles, and their final shape depends on which parts of the rendered scene actually changed.
This distinction is important when tuning the two groups together:
```text
picture content
picture tiles
invalidation / retained rendering
changed regions
update rectangles
partial presentation / compositor
Present1
```
Smaller tiles can provide finer-grained invalidation and reduce the amount of content that needs to be rebuilt when only a small region changes, but they can also increase bookkeeping and the number of regions involved.
Conversely, allowing more update rectangles gives the compositor more freedom to preserve multiple independent changes instead of merging them into fewer, potentially larger update regions, but increases the amount of metadata and processing required to manage those regions.
They should therefore be tuned together, but they are **not interchangeable**:
> **Tile size controls the granularity of rendered content; update-rect limits control how much of that granularity can survive through the partial-update/presentation path.**
For this reason, a `512×512` picture-tile configuration does not imply a particular number of update rectangles. The two values describe different stages of the pipeline and should be optimized according to the workload rather than mathematically matched to one another.


### 3a. Rects and Surface pool size: 
By default WebRender is configured to generate and push frames/layers into the vram, an entire new **tile** everytime something change inside the page but if a lot of changes happens at the same time, an entire new frame must be redrawn. 
In order to optimize this behaviour, we need to dig into about:config and look for the strings to modify : 
* gfx.webrender.compositor.max_update_rects
* gfx.webrender.max-partial-present-rects 
* gfx.webrender.compositor.surface-pool-size

These parameters **limit how many update regions can be represented and processed as partial updates** before WebRender/compositing has to fall back to a less granular update.
Instead of requiring a large contiguous region to be updated, **partial presentation** can preserve smaller independent update regions:
* Only the **changed portion** (tile) of the page is updated
* Memory traffic is drastically reduced
* CPU workload becomes more efficient

**What Happens If the Pool Is Too Small ?**
Example:
* max_update_rects = 1
* max-partial-present-rects = 1
* surface-pool-size = 1 

in this case if more than 1 region need updating simultaneously:
* WebRender runs out of available surfaces
* It must either allocate new ones or wait for reuse
* The compositor pipeline stalls temporarily

**This can lead to:**
* micro-stuttering
* increased CPU usage
* additional memory allocation overhead

**What Happens If the rect or Pool Is Too Large ?**
Example:
* max_update_rects = 1024
* max-partial-present-rects = 1024
* surface-pool-size = 1024

**This will not break rendering, but will introduces:**
* abnormal overhead
* larger memory footprint
* more internal bookkeeping
* no meaningful performance gain
(Basically beyond a certain point, increasing both rects and the pool size only *waste resources*.)

**Wich value should i set then ?**
A moderate value is generally preferable. On a CPU-rendered system, values such as 8–16 provide enough granularity for multiple independent updates without creating excessive compositor bookkeeping:
* max_update_rects = 16
* max-partial-present-rects = 16
while instead, for the pool size, the sweet spoot inbetween memory footprint and reasonable necessity is around 4 times the above tuned parameter to compensate frame overlap and reuse latency :
* surface-pool-size = 64

**So what if i have a threadripper ? 128 threads equal to 128 rects ?** not exactly ! Keep using a moderate value (typically 2–16), increasing beyond that rarely improves performance in CPU rendering mode and will introduce overhead)

### 3b. Tiles:
WebRender divides rendered picture content into tiles, while the compositor can represent changes as update rectangles.

**Too granular vs too large:**
* Too small tiles / too many rects: precise updates but more scheduling and CPU overhead.
* Too large tiles / too few rects: inefficient, large areas rasterized unnecessarily, more RAM usage.
```
gfx.webrender.picture-tile-height	= 512		
gfx.webrender.picture-tile-width	= 512
```
* Tile size tuning depends on screen resolution, page complexity, and workload.
* A tile size of 512x512 is often a **good balance** for picture tiles.
* Blob tiles around 256px are efficient for most text-heavy pages.
* A higher update-rect limit allows more independent partial updates to be retained instead of forcing them into fewer, larger update regions.
  
**Smaller (more granular):**
* More precise updates
* Higher cache efficiency for small changes
* Increased CPU overhead (scheduling, bookkeeping)
* Potentially higher memory usage
  
**Larger tiles:**
* Fewer tiles to manage
* Lower CPU overhead
* More unnecessary rasterization (overdraw)
* Less efficient for dynamic content

### 3c. Blob Tiles:
Blob tiles are used when WebRender rasterizes blob content such as text and vector graphics.
Blobs are cached separately from picture tiles to avoid re-rasterizing vector graphics repeatedly.
Smaller blob tiles increase cache hits for small vector updates but consume more bookkeeping resources.
```
gfx.webrender.blob-tile-size = 256
```
* With 256, blob content is rasterized in 256×256 tiles.”.
* Default value is often ok-ish.

**Smaller blob:**
* Can improve cache reuse for small updates.
* Reduced re-rasterization of text
* Higher CPU overhead (more tiles to manage)
  
**Larger blob:**
* Less bookkeeping
* More redundant rasterization
* Less efficient for dynamic text-heavy content

### 3d. Text Rendering Optimization (Blob Tiles + Font Cache):
Text rendering in WebRender relies on a combination of blob images and glyph-level caching. Understanding how these interact is critical for optimizing CPU-based rendering performance.
Blob tiles handle large-scale rasterization of text regions (entire paragraphs, UI blocks, vector content).
Font (glyph) cache stores individual character shapes (glyph bitmaps reused across frames).

**Rendering pipeline (simplified):**
1 Text is converted into vector shapes
2 Shapes are grouped into blob images
3 Blob images are rasterized into tiles
4 Individual glyphs are cached separately in the font cache

**Why this matters:**
* Blob tiles avoid re-rasterizing entire text regions
* Font cache avoids re-rasterizing individual glyphs
* Together, they significantly reduce CPU workload during scrolling and updates

**Key interaction:**
* Blob tiles operate at a macro level (text blocks)
* Font cache operates at a micro level (characters)
  
If both are well configured, Text rendering becomes **highly cache-efficient**, especially during scrolling

Recommended configuration (CPU rendering):
```
gfx.content.skia-font-cache-size = 32–64
```

**Notes:**
Blob caching reduces large-scale raster work
Font cache reduces fine-grained glyph work
Performance gains are most visible in CPU rendering mode
Efficient text rendering is achieved by combining **coarse-grained caching** (blob tiles) and **fine-grained caching** (glyph cache)
Balancing both levels is key to minimizing CPU load and improving smoothness

### 3e. Disabling ClearType:
ClearType is the default text smoothing technique on Windows, based on subpixel rendering.
It improves text clarity by using the RGB subpixel structure of LCD displays.
While visually effective, it introduces additional processing overhead during text rasterization.

**How ClearType works:**
Instead of rendering text using whole pixels, each pixel is split into subpixels (R, G, B).
Glyph edges are calculated at subpixel precision.
Additional filtering and blending is applied.
(This results in sharper text and better readability at small sizes)

**Why it adds overhead:**
ClearType increases rendering complexity because adds more calculations per glyph (subpixel precision), additional blending operations resulting in more complex rasterization pipeline.
In CPU rendering mode, this translates into a higher cost per glyph so more CPU usage during text rendering.

**You can reduce this overhead by switching to simpler rendering modes:**
```
gfx.font_rendering.cleartype_params.rendering_mode = 0
gfx.font_rendering.cleartype_params.cleartype_level = 0
```
**What happens when disabled ?**
Without ClearType, Firefox falls back to simpler methods:
grayscale anti-aliasing or full pixel aliasing (depending on machine configuration).

**Pros:**
* Lower CPU usage
* Faster glyph rasterization
* Smoother scrolling in text-heavy pages

**Cons:**
* Text appears less sharp
* Reduced readability at small font sizes
* Loss of subpixel precision

**When to use this tweak ?**
CPU-only rendering setups, low-power systems, performance-critical scenarios, text-heavy scrolling workloads.

**When is not recommended ?**
high-DPI displays where clarity matters or users sensitive to text sharpness.

### 3f. Retaining Display List for Stacking Contexts:
By default, Firefox may not retain display lists for stacking contexts.  
Stacking contexts are groups of elements that WebRender treats as separate sub-scenes (e.g., due to z-index, opacity, transform, filters).
Frequent invalidation of stacking contexts can force repeated rebuilds of display lists, reducing cache effectiveness and increasing CPU workload.
Keeping stacking contexts retained allows WebRender to reuse display lists across frames, which improves performance in complex UIs, especially in CPU rendering mode:
```
layout.display-list.retain.sc = true
```
This setting helps ensure that display lists for stacking contexts are kept alive and reused, reducing unnecessary rebuilds and lowering CPU overhead.

### 3g. Thread‑Local Arenas for WebRender Threads:
Firefox’s memory allocator (jemalloc) supports thread‑local arenas which can reduce lock contention when threads do most allocations and deallocations on themselves.
WebRender exposes two preferences to optionally enable dedicated arenas for specific internal threads:
```
gfx.webrender.frame-builder-thread-local-arena 
gfx.webrender.scene-builder-thread-local-arena
```
These prefs control whether a dedicated thread‑local arena is used for allocations on the frame builder and scene builder threads.
A thread‑local arena can reduce contention for locks during allocations when that thread is the primary user of its arena.
By default these are false (disabled), so enabling them can reduce allocation bottlenecks during scene building and frame construction.

**Note:** Thread‑local arenas do not change the fundamental rendering pipeline; they optimize memory allocation patterns. Their benefits are most noticeable under heavy parallel allocation load, and behavior depends on the specific workload and threading pattern in WebRender.
Thread‑local arenas help reduce allocator contention but are not a substitute for good cache locality and reduced invalidation.

### 3h. Worker Threads:
WebRender uses a pool of worker threads for raster and related tasks. The *gfx.webrender.workers* preference controls the maximum number of worker threads that WebRender will create:
```
dom.workers.maxPerDomain = (number of your cpu core/threads)
```
By default this value is 512 but that number is only an upper bound.
Firefox internally limits the actual worker count based on available hardware and other constraints.
Simply increasing this value alone does not proportionally increase parallelism and can introduce bookkeeping overhead if not paired with an appropriate workload.
In practice, tuning this value to reflect the number of available core's threads could potentially yields better CPU utilization without excessive contention or overhead.
This reduces scheduling overhead and aligns the worker pool with actual CPU capacity, which can help avoid excessive thread management costs in CPU‑bound rendering.
Worker thread count should align with system capabilities and overall rendering workload. Too many threads can degrade performance due to overhead.


### 3i. WebRender Software D3D11 Upload Mode:
When using software WebRender, avoiding synchronization stalls can be more important than minimizing every redundant operation.
```
gfx.webrender.software.d3d11.upload-mode = 3  (default 4 on FF153)
```
Mode `3` favors keeping the rendering/presentation pipeline moving instead of waiting unnecessarily for previous operations to complete.
This is particularly useful for:
* rapid scrolling
* dynamically changing pages
* software WebRender configurations
* workloads where CPU rasterization is fast enough that synchronization becomes the larger bottleneck

The philosophy is simple:
> **Prefer a little redundant work over stalling the pipeline.**
This works well alongside the other software WebRender optimizations because it does not try to reduce the amount of work at all costs; it reduces the likelihood that otherwise independent rendering work gets serialized by synchronization.
Benchmark against the default value on your own workload. The target is smoother frame delivery and fewer stalls, not necessarily lower total rendering work.

### 3l. WebRender Batching Lookback:
WebRender's batching system searches previous primitives for opportunities to merge compatible rendering operations.
```
gfx.webrender.batching.lookback = 40  (default 10 on FF153)
```
A higher lookback gives the batch builder more opportunities to find compatible primitives instead of immediately creating another batch.
The trade-off is a small increase in CPU-side batch construction work.
For software WebRender, this can be worthwhile because CPU rasterization is often fast enough that reducing unnecessary rendering operations is preferable to minimizing the batcher's own bookkeeping.
```
smaller lookback : less searching : fewer batching opportunities

larger lookback : more searching : more batching opportunities
```
This is **not** a frame queue, synchronization depth, or latency setting. It is a batching heuristic.
Together with the other WebRender tuning parameters, the goal is to favor efficient continuous rendering rather than aggressively minimizing every individual operation.


### 3z. Gradients:
Some websites make heavy use of gradient calculations.
These operations can be expensive in software rendering.
Disabling precise gradient calculations reduces CPU usage while keeping visual differences negligible.
```
gfx.webrender.precise-conic-gradients = false		
gfx.webrender.precise-conic-gradients-swgl = false		
gfx.webrender.precise-linear-gradients-swgl = false		
gfx.webrender.precise-radial-gradients = false		
gfx.webrender.precise-radial-gradients-swgl = false
```

---

## 4. Beyond barebone optimizations : why web browsers are mainly single-threaded ?
* Curious to know more about software renderers still beat gpu acceleration ? have a look **[here](https://github.com/aston89/Firefox-Waterfox-Mercury-Webrender-optimization-guide/blob/main/DOM_SINGLE_THREADED.md)**
* Curious to know more about browsers rendering pipelines ? have a look **[here](https://github.com/aston89/Firefox-Waterfox-Mercury-Webrender-optimization-guide/blob/main/RENDERING_PIPELINE.md)**
* curious to know more about GPU layers ? have a look **[here](https://github.com/aston89/Firefox-Waterfox-Mercury-Webrender-optimization-guide/blob/main/GPU_LAYERS.md)**

