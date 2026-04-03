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

## 3. The Critical Optimization:

### 3a. "Rects" and "Surface pool size": 
By default WebRender is configured to generate and push frames/layers into the vram, an entire new **tile** everytime something change inside the page but if a lot of changes happens at the same time, an entire new frame must be redrawn. 
In order to optimize this behaviour, we need to dig into about:config and look for the strings to modify : 
* gfx.webrender.compositor.max_update_rects
* gfx.webrender.max-partial-present-rects 
* gfx.webrender.compositor.surface-pool-size

This will let **a X of independents concurrent updates of tile regions**.
Instead of rewriting the entire frame in *RAM*:
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
It's recommended to use a value equal to the number of your cpu cores/thread for example if you have 8 core and 16 thread (hyperthreading) it's advisable to set :
* max_update_rects = 16
* max-partial-present-rects = 16
while instead, for the pool size, the sweet spoot inbetween memory footprint and reasonable necessity is around 4 times the above tuned parameter to compensate frame overlap and reuse latency :
* surface-pool-size = 64

**So what if i have a threadripper ? 128 threads equal to 128 rects ?** not exactly ! Keep using a moderate value (typically 2–16), increasing beyond that rarely improves performance in CPU rendering mode and will introduce overhead)

### 3b. "Tiles":
WebRender divides the page into tiles and updates rects to efficiently rasterize only the changed parts of a page. This helps reduces CPU workload and optimizes memory usage while keeping pages responsive and by default is set to portions of 512x512 tiles.

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
* Increasing max_update_rects allows multiple small updates to happen in a frame without collapsing into large redraws.
  
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

### 3c. "Blob Tiles":
The tile size for rasterizing complex text and vector shapes are called "blobs".
Blobs are cached separately from picture tiles to avoid re-rasterizing vector graphics repeatedly.
Smaller blob tiles increase cache hits for small vector updates but consume more bookkeeping resources.
```
gfx.webrender.blob-tile-size = 256
```
* Each blob tile is 256x256 pixels.
* Default value is often ok-ish.

**Smaller blob:**
* Better cache reuse for small updates
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

### 3e. Disabling ClearType
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

---

## 3e. Gradients:
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

