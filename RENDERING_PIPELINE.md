## Browser Rendering Pipeline (Simplified)

Understanding how a browser renders a page is essential to understanding where performance bottlenecks actually occur.

Even though modern browsers contain many parallel subsystems, the **core rendering pipeline remains sequential at key stages**.

This document describes the high-level pipeline used by modern browsers, with particular focus on WebRender-based engines.

---

## High-Level Rendering Pipeline

A typical rendering cycle looks like this:

```
JavaScript
   ↓
DOM mutation
   ↓
Style recalculation
   ↓
Layout
   ↓
Display list generation
   ↓
Rasterization
   ↓
Compositing / Presentation
```

Each stage transforms the page state into a progressively more concrete representation until a final frame can be displayed.

---

## 1. JavaScript Execution

JavaScript runs on the **main thread**.

During execution it may:

* modify the DOM
* change CSS classes
* update inline styles
* insert or remove elements
* trigger animations
* respond to user events

Any of these actions may invalidate parts of the page.

---

## 2. DOM Mutation

The DOM is the **structural representation of the page**.

Operations include:

* inserting nodes
* removing nodes
* changing attributes
* modifying text content

These mutations mark parts of the page as **dirty**, meaning they must be recomputed during the next rendering pass.

---

## 3. Style Recalculation

Once the DOM changes, the browser must determine which CSS rules apply to each element.

This stage resolves:

* CSS selectors
* inheritance
* computed styles
* media queries
* pseudo elements

Only elements affected by the DOM mutation need to have their styles recalculated.

---

## 4. Layout (Reflow)

Layout determines the **geometry of the page**.

The engine calculates:

* element positions
* element sizes
* line wrapping
* box model calculations
* flow relationships

Layout can be expensive because many elements depend on the dimensions of others.

For example:

* flexbox containers
* grid layouts
* percentage-based sizing
* dynamic text wrapping

---

## 5. Display List Generation

After layout, the browser converts the page into a **display list**.

A display list is essentially a list of drawing commands such as:

```
draw rectangle
draw border
draw text
draw image
apply transform
apply clip
```

This list describes *what needs to be drawn*, but not yet *how it will be rasterized*.

---

## 6. Rasterization

Rasterization converts display list commands into **actual pixels**.

Depending on configuration, this may happen:

* on the GPU
* on the CPU
* in hybrid pipelines

In WebRender software mode, rasterization happens on the **CPU**, typically using tile or rect-based updates.

Only the parts of the frame that changed need to be rasterized again.

---

## 7. Compositing and Presentation

The final step is presenting the frame to the display.

This may involve:

* combining multiple surfaces
* applying transforms
* blending layers
* presenting the framebuffer

Even when rendering is done in software, the final frame is usually presented through a graphics API such as **Direct3D**, **OpenGL**, or **Metal**.

---

## Incremental Rendering

Modern browsers avoid redrawing the entire page whenever possible.

Instead they update only **regions of the frame that changed**.

This is typically done using:

* tiles
* update rectangles
* partial frame updates

Efficient partial updates are critical for good performance, especially on pages with frequent dynamic changes.

---

## Where Performance Problems Usually Occur

Most real-world performance issues occur in one of these areas:

**Main thread bottlenecks**

* large DOM mutations
* heavy JavaScript execution
* layout thrashing

**Rendering overhead**

* excessive layer compositing
* frequent texture uploads
* inefficient invalidation regions

**Memory pressure**

* large surface allocations
* excessive buffering
* large DOM trees

Understanding the rendering pipeline helps identify which subsystem is actually responsible for slowdowns.

---

## Key Takeaway

While browsers contain many complex subsystems, the rendering process still follows a clear sequence:

```
DOM → Style → Layout → Display List → Rasterization → Presentation
```

Optimizing performance often means **reducing the amount of work performed in these stages**, especially on the main thread.
