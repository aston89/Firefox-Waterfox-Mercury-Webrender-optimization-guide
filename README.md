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

## 2. The Alternative: Software WebRender

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

## 3. The Critical Optimization: "Update Rectangles" and "surface pool size"

By default WebRender is configured to generate and push, into the vram, an entire new frame everytime something change inside the page. 
In order to optimize this behaviour, we need to dig into about:config and look for two strings to modify : 
* gfx.webrender.compositor.max_update_rects = 128
* gfx.webrender.compositor.surface-pool-size = 128

This will creates **128 independent update regions with correlated buffers**.
Instead of rewriting the entire frame in VRAM:
* Only the **changed portion** of the page is updated
* Memory traffic is drastically reduced
* CPU workload becomes minimal

**What Happens If the Pool Is Too Small ?**
Example:
* max_update_rects = 128
* surface-pool-size = 8

If more than eight regions need updating simultaneously:
WebRender runs out of available surfaces
It must either allocate new ones or wait for reuse
The compositor pipeline stalls temporarily

**This can lead to:**
* micro-stuttering
* increased CPU usage
* additional memory allocation overhead

**What Happens If the Pool Is Too Large ?**
Example:
* surface-pool-size = 512

This will not break rendering, but will introduces:
* unnecessary overhead
* larger memory footprint
* more internal bookkeeping
* no meaningful performance gain

Basically beyond a certain point, increasing both **rects** and the **pool size** only waste resources.

---

## 4. Optional Fine Tuning

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

# Beyond barebone optimizations : why web browsers are mainly single-threaded
Curious to know more about software renderers still beat gpu acceleration ? have a look **[here](https://github.com/aston89/Firefox-Waterfox-Mercury-Webrender-optimization-guide/blob/main/DOM-SINGLE-THREADED.md)**.
Curious to know more about browsers rendering pipelines ? have a look **[here](https://github.com/aston89/Firefox-Waterfox-Mercury-Webrender-optimization-guide/blob/main/RENDERING_PIPELINE.md)**

