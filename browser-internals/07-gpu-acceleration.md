# 07 — GPU Acceleration

> **"The GPU was built to move pixels fast — billions of them per second. The browser uses it for compositing, rasterization, and WebGL. Understanding when the GPU is helping and when it's hurting is the difference between fluid animations and mysterious performance regressions."**

GPU acceleration is one of the most misunderstood concepts in browser performance. Developers add `transform: translateZ(0)` hoping it "makes things faster," without understanding what the GPU actually does, when it helps, and critically — when it makes things worse. This document covers the GPU's role in the browser pipeline, how hardware acceleration works, what triggers it, and the real tradeoffs between CPU and GPU rendering paths.

---

## 📚 Table of Contents

1. [CPU vs GPU — The Fundamental Difference](#1-cpu-vs-gpu--the-fundamental-difference)
2. [What the GPU Does in the Browser](#2-what-the-gpu-does-in-the-browser)
3. [The GPU Rendering Pipeline](#3-the-gpu-rendering-pipeline)
4. [Hardware-Accelerated Compositing](#4-hardware-accelerated-compositing)
5. [GPU Rasterization](#5-gpu-rasterization)
6. [WebGL and Direct GPU Access](#6-webgl-and-direct-gpu-access)
7. [GPU Memory Architecture](#7-gpu-memory-architecture)
8. [When GPU Acceleration Helps](#8-when-gpu-acceleration-helps)
9. [When GPU Acceleration Hurts](#9-when-gpu-acceleration-hurts)
10. [The Texture Upload Bottleneck](#10-the-texture-upload-bottleneck)
11. [GPU-Accelerated CSS Filters](#11-gpu-accelerated-css-filters)
12. [Checking GPU Usage](#12-checking-gpu-usage)
13. [GPU Acceleration on Mobile](#13-gpu-acceleration-on-mobile)
14. [Good Practices](#14-good-practices)
15. [Bad Practices](#15-bad-practices)
16. [Common Mistakes](#16-common-mistakes)
17. [Interview-Level Explanation](#17-interview-level-explanation)
18. [Exercises](#18-exercises)

---

## 1. CPU vs GPU — The Fundamental Difference

Understanding GPU acceleration requires understanding the fundamental architectural difference between CPUs and GPUs:

```
CPU (Central Processing Unit):
  ┌─────────────────────────────────────────────────────┐
  │  4–32 large, powerful cores                          │
  │  Each core: complex instruction set, deep caches,    │
  │             branch prediction, out-of-order execution│
  │  Best at: sequential logic, branching, single-thread │
  │           performance                                 │
  │  RAM: gigabytes of system memory (DDR5)              │
  └─────────────────────────────────────────────────────┘

GPU (Graphics Processing Unit):
  ┌─────────────────────────────────────────────────────┐
  │  Thousands of small, simple cores (shaders)          │
  │  Each core: simple instruction set, minimal caching, │
  │             no branch prediction                     │
  │  Best at: massively parallel operations on data,     │
  │           same operation applied to thousands of     │
  │           pixels simultaneously                      │
  │  VRAM: dedicated video memory (GDDR6) — fast for    │
  │        GPU access, slow for CPU access               │
  └─────────────────────────────────────────────────────┘
```

### The Pixel Parallel Problem

Rendering pixels is inherently parallelizable:

```
Rendering a 1920×1080 screen = 2,073,600 pixels
Each pixel can be computed independently
(what color pixel (x=0,y=0) should be doesn't affect pixel (x=1000,y=500))

CPU approach: compute pixels serially
  4 cores × 2 threads × ? fps = limited throughput

GPU approach: compute pixels in parallel
  4096 shader cores × 2,073,600 pixels =
  Can process the entire screen ~2× per shader batch
  → Hundreds of frames per second possible
```

This is why the GPU is vastly superior for rendering — the workload is embarrassingly parallel.

---

## 2. What the GPU Does in the Browser

The browser uses the GPU for three main tasks:

```
┌────────────────────────────────────────────────────────────────┐
│               BROWSER GPU USAGE                                 │
│                                                                  │
│  1. COMPOSITING                                                 │
│     Merging layer bitmaps into the final screen image           │
│     GPU: Apply transforms, opacity, blend modes to textures     │
│     Always GPU (in hardware-accelerated browsers)               │
│                                                                  │
│  2. GPU RASTERIZATION                                           │
│     Converting paint display lists into pixel bitmaps           │
│     GPU: Fill geometry with colors (faster than CPU for simple  │
│          rectangles, solid fills, gradients)                    │
│     Optional (Chrome uses GPU rasterization by default)         │
│                                                                  │
│  3. WEBGL / WEBGPU                                              │
│     Direct GPU programming via JavaScript                        │
│     Full access to shader pipeline                              │
│     Used for: 3D graphics, games, complex visualizations,       │
│              ML inference (TensorFlow.js), image processing     │
└────────────────────────────────────────────────────────────────┘
```

---

## 3. The GPU Rendering Pipeline

When the browser submits a frame to the GPU, it goes through a shader pipeline:

### OpenGL/Metal/Vulkan Pipeline (Simplified)

```
CPU sends to GPU:
  - Vertex data (layer rectangles as geometry)
  - Texture data (layer bitmaps)
  - Shader programs (how to draw them)
  - Transform matrices (from CSS transforms)

GPU pipeline:
  ┌─────────────────┐
  │  Vertex Shader  │  → Transforms vertex positions using matrices
  └────────┬────────┘    (applies CSS transforms: translate, scale, rotate)
           │
  ┌────────▼────────┐
  │   Rasterizer    │  → Determines which pixels are inside the geometry
  └────────┬────────┘
           │
  ┌────────▼────────┐
  │ Fragment Shader │  → Colors each pixel (samples from texture)
  └────────┬────────┘    (applies opacity, blend modes, filters)
           │
  ┌────────▼────────┐
  │  Blend/Output   │  → Merges with existing frame buffer (alpha blending)
  └────────┬────────┘
           │
  ┌────────▼────────┐
  │  Frame Buffer   │  → Final pixel data displayed on screen
  └─────────────────┘
```

### How CSS `transform` Maps to GPU

```javascript
// CSS: transform: translate(100px, 50px) scale(1.2) rotate(45deg)

// GPU receives a transformation matrix:
// [ cos(45°)×1.2  -sin(45°)×1.2  100 ]
// [ sin(45°)×1.2   cos(45°)×1.2   50 ]
// [      0              0           1 ]

// Vertex shader applies this matrix to layer's corner vertices
// GPU does this in hardware — essentially free
// No repainting, no re-rasterization — just matrix math on existing bitmap
```

This is why `transform` is "free" — the GPU can apply matrix transformations to existing textures in microseconds, without touching the CPU or repainting any pixels.

---

## 4. Hardware-Accelerated Compositing

Hardware-accelerated compositing is the browser feature that enables compositor layers to be merged by the GPU rather than the CPU.

### Software Compositing (Old Browsers)

```
CPU compositing:
  1. CPU copies layer bitmaps into a single output buffer
  2. For each pixel: manually blend source and destination
  3. Write result to screen buffer
  4. Copy to screen

Performance: O(total_pixels × num_layers) — slow for many overlapping layers
CPU usage: high, blocks main thread
```

### Hardware Compositing (Modern Browsers)

```
GPU compositing:
  1. Layer bitmaps uploaded to GPU as textures (once, cached)
  2. GPU renders quads (rectangles) textured with each layer
  3. GPU applies transforms, opacity, and blend modes in shaders
  4. GPU outputs directly to frame buffer (screen)

Performance: O(num_layers) with GPU parallelism — very fast
CPU usage: near zero for compositing
```

### The Compositing Sequence

```
Main thread → Compositor thread → GPU

Main thread:
  - Generates display list (what to paint)
  - Sends "commit" to compositor

Compositor thread:
  - Receives commit
  - Sends dirty tiles to raster threads
  - After rasterization: uploads new bitmaps to GPU as textures
  - Sends "draw quad" commands to GPU:
    "Draw texture A at position (0,0) with transform M and opacity 1.0"
    "Draw texture B at position (0,60) with transform M2 and opacity 0.9"
    ...

GPU:
  - Executes draw commands using vertex + fragment shaders
  - Outputs final frame to display
```

---

## 5. GPU Rasterization

Beyond compositing, Chrome uses the GPU to also **rasterize** (convert paint commands to pixels). This is the Skia GPU backend (Ganesh), and more recently the new Skia replacement called Graphite.

### CPU Rasterization (Software)

```
CPU rasterization:
  Input: paint display list (fill rect, draw text, draw image, etc.)
  Process: CPU executes drawing commands, writes pixels to memory
  Output: bitmap in RAM
  Then: uploaded to GPU as texture for compositing

Advantages:
  - Works everywhere (no GPU dependency)
  - Handles complex/arbitrary paths well
  - Predictable behavior

Disadvantages:
  - CPU-bound — competes with JavaScript for CPU time
  - Slow for large areas or complex operations
  - Bandwidth: must upload bitmaps from RAM to VRAM
```

### GPU Rasterization (Hardware)

```
GPU rasterization:
  Input: paint display list
  Process: GPU's vertex/fragment shaders execute drawing commands
  Output: bitmap directly in VRAM (no upload needed!)
  Then: directly available as texture for compositing

Advantages:
  - GPU-bound — doesn't compete with JavaScript
  - Fast for fills, rectangles, simple geometry
  - No RAM→VRAM upload bottleneck (bitmap already in VRAM)
  - Parallelism: many tiles rasterized simultaneously

Disadvantages:
  - Complex paths (intricate SVG, anti-aliased curves) can be slower
  - GPU memory consumption
  - Less support on older hardware
```

### Enabling/Checking GPU Rasterization

```
chrome://flags/#enable-gpu-rasterization
(Enabled by default in Chrome for supported hardware)

To verify in DevTools:
  Performance tab → Record → Stop
  → Bottom panel: "Raster" entries
  → "GPU Raster" = GPU-accelerated rasterization active
  → CPU raster = software fallback

Or: chrome://gpu
  → "Rasterization: Hardware accelerated" ← GPU raster
  → "Rasterization: Software only"        ← CPU fallback
```

---

## 6. WebGL and Direct GPU Access

WebGL gives JavaScript direct access to the GPU's shader pipeline through the OpenGL ES 2.0/3.0 API. WebGPU is the modern replacement with a cleaner API.

### WebGL Usage

```javascript
const canvas = document.getElementById("canvas");
const gl = canvas.getContext("webgl"); // or 'webgl2'

// 1. Write shaders (GLSL — GPU programs)
const vertexShaderSrc = `
  attribute vec2 a_position;
  void main() {
    gl_Position = vec4(a_position, 0.0, 1.0);
  }
`;

const fragmentShaderSrc = `
  precision mediump float;
  uniform vec4 u_color;
  void main() {
    gl_FragColor = u_color;
  }
`;

// 2. Compile and link shaders
const vertexShader = gl.createShader(gl.VERTEX_SHADER);
gl.shaderSource(vertexShader, vertexShaderSrc);
gl.compileShader(vertexShader);

const fragmentShader = gl.createShader(gl.FRAGMENT_SHADER);
gl.shaderSource(fragmentShader, fragmentShaderSrc);
gl.compileShader(fragmentShader);

const program = gl.createProgram();
gl.attachShader(program, vertexShader);
gl.attachShader(program, fragmentShader);
gl.linkProgram(program);
gl.useProgram(program);

// 3. Send geometry to GPU
const vertices = new Float32Array([
  0.0,
  0.5, // top
  -0.5,
  -0.5, // bottom-left
  0.5,
  -0.5, // bottom-right
]);

const buffer = gl.createBuffer();
gl.bindBuffer(gl.ARRAY_BUFFER, buffer);
gl.bufferData(gl.ARRAY_BUFFER, vertices, gl.STATIC_DRAW);

// 4. Configure vertex shader input
const positionLoc = gl.getAttribLocation(program, "a_position");
gl.enableVertexAttribArray(positionLoc);
gl.vertexAttribPointer(positionLoc, 2, gl.FLOAT, false, 0, 0);

// 5. Set color uniform
const colorLoc = gl.getUniformLocation(program, "u_color");
gl.uniform4f(colorLoc, 0.0, 0.5, 1.0, 1.0); // RGBA

// 6. Draw
gl.clearColor(0, 0, 0, 1);
gl.clear(gl.COLOR_BUFFER_BIT);
gl.drawArrays(gl.TRIANGLES, 0, 3);
// GPU renders 1 blue triangle — 100% on GPU
```

### When to Use WebGL vs CSS vs Canvas 2D

```
Canvas 2D (CPU):
  Use for: data charts (< 10,000 points), image processing,
           text rendering, simple animations with complex logic
  Limit: ~100,000 draw operations/frame before slowdown

CSS/SVG:
  Use for: UI animations, small number of elements,
           interactive elements (hover, click)
  Limit: ~1,000 elements before SVG gets slow

WebGL/WebGPU:
  Use for: 3D rendering, particle systems (millions of particles),
           shader effects, data visualization at extreme scale,
           image processing pipelines, ML inference
  Limit: GPU memory and shader complexity
```

---

## 7. GPU Memory Architecture

Understanding where GPU memory comes from and how it's managed is critical for understanding performance:

### Memory Types

```
DESKTOP (dedicated GPU):
  System RAM (DDR5): 16-128GB
    └── CPU reads/writes here
    └── JavaScript heap lives here

  VRAM (GDDR6): 4-24GB (dedicated GPU memory)
    └── GPU reads/writes here
    └── Compositor layer textures live here
    └── WebGL textures/buffers live here
    └── Fast for GPU, slow for CPU

  PCIe Bus (CPU ↔ GPU):
    └── Bandwidth: 16-64 GB/s
    └── Bottleneck for texture uploads

MOBILE (integrated GPU — shares memory):
  Unified Memory: 4-16GB (shared between CPU and GPU)
    └── Both CPU and GPU use the same physical RAM
    └── No explicit texture upload (no PCIe bus)
    └── But: total budget shared — GPU textures compete with RAM
    └── Memory pressure affects both CPU and GPU
```

### Memory Bandwidth Implications

```
Texture upload (CPU → GPU):
  Desktop: Fast (PCIe bandwidth ~32GB/s) but not free
  Mobile: No physical transfer but still costs time (cache coherency)

Large texture upload (e.g., 1920×1080 RGBA = 8MB):
  Desktop: ~0.25ms to upload via PCIe
  Mobile: near-zero (unified memory) but marks cache lines dirty

Uploading 100 large textures per second:
  Desktop: 800MB/s PCIe bandwidth → significant
  Mobile: cache coherency overhead → significant
```

---

## 8. When GPU Acceleration Helps

### ✅ Many Layers Being Composited

```
Scenario: App with 10 overlapping layers (header, modal, tooltips, etc.)

Without GPU compositing (CPU):
  CPU blends all layers pixel-by-pixel = O(pixels × layers) = slow

With GPU compositing:
  GPU blends all layers using shaders = parallelism
  Speed: same regardless of layer count (up to GPU limits)

Winner: GPU for compositing
```

### ✅ Large Area Transforms

```css
/* Moving a large element using transform */
.hero-image {
  width: 1920px;
  height: 1080px;
  transform: translateX(var(--offset));
  will-change: transform;
}

/* GPU: moves the texture's transform matrix — no pixels recomputed */
/* Cost: O(1) regardless of element size */

/* CPU equivalent (top/left): layout + paint + composite */
/* Cost: O(pixels) — more expensive as element gets larger */
```

### ✅ Opacity and Blend Mode Animations

```
Opacity animation on a compositor layer:
  GPU adjusts alpha channel of texture in fragment shader
  Cost: O(1) per frame — trivial GPU operation

vs. CPU without compositor layer:
  CPU must repaint underlying content
  Cost: O(visible_pixels) — expensive
```

### ✅ WebGL Visualizations at Scale

```javascript
// 1,000,000 particles animated per frame
// CPU: would take 33ms+ = freezes the page

// GPU (vertex shader): transforms all 1M vertices in parallel
// Cost: ~1ms on modern GPU
// Main thread: free to handle UI while GPU renders
```

---

## 9. When GPU Acceleration Hurts

GPU acceleration is not free. These scenarios show where it degrades performance:

### ❌ Texture Upload Overhead

```javascript
// ❌ Animating a property that requires re-rasterization
element.style.backgroundColor = newColor; // repaint needed
// 1. CPU repaints element into bitmap in RAM
// 2. Upload bitmap from RAM to VRAM (GPU memory)
// 3. GPU composites the updated texture

// The RAM→VRAM upload is the bottleneck for frequently-updated layers
// On desktop: ~0.3ms per large texture upload
// Doing this 60 times/second: 18ms/s just for uploads!
```

### ❌ Too Many Layers — GPU Memory Exhaustion

```
Scenario: 1000 list items with will-change: transform
  Each item: 800×100px = 80,000 pixels × 4 bytes = 320KB
  1000 items: 320MB of GPU memory

Mobile GPU (1-2GB shared): catastrophic — app performance crashes
```

### ❌ Texture Thrashing

When too many textures compete for limited GPU memory:

```
GPU memory limit reached:
  Browser evicts old textures to free space
  On scroll: evicted tiles need to be re-uploaded
  Result: checkerboard pattern, scroll jank, dropped frames

This happens when:
  - Too many compositor layers
  - Layer bitmaps are very large
  - WebGL uses large textures
  - Lots of high-DPI content
```

### ❌ High-DPI (Retina) Layer Cost

```
On a 2× Retina display:
  Logical 1920×1080 display
  Physical 3840×2160 pixels

Compositor layer for a full-page element:
  1920×1080 × (2)² = 4× more pixels = 4× more GPU memory

Full-page layer at 2× DPI: ~32MB per layer
vs 1× DPI: ~8MB per layer

Mobile with Retina display: layer memory costs are multiplied
This is why mobile GPU memory pressure is critical
```

---

## 10. The Texture Upload Bottleneck

One of the most overlooked GPU performance problems is the **texture upload bottleneck** — moving bitmap data from CPU RAM to GPU VRAM.

### When Uploads Happen

```
Texture uploads happen when:
  1. A compositor layer is first created (initial upload)
  2. A compositor layer is repainted (content changed → re-upload)
  3. A tile is evicted from GPU memory and needs to be re-uploaded

Uploads do NOT happen for:
  - transform changes (matrix math only)
  - opacity changes (shader math only)
  - scroll (layer positions change, no re-upload)
```

### Measuring Upload Cost

```javascript
// In Chrome Performance timeline:
// "Commit" events on the compositor thread include texture uploads
// Look for long "Commit" durations after paint events

// Rough upload bandwidth: ~20-100 GB/s (PCIe Gen4 × 16 lanes)
// 8MB texture (1920×1080 RGBA): ~0.08-0.4ms to upload
// If animating background-color at 60fps: 60 × 0.2ms = 12ms/s wasted
```

### Minimizing Uploads

```javascript
// ✅ Avoid repainting compositor layers
// If content must change, change only compositor-friendly properties:
element.style.transform = "..."; // no upload
element.style.opacity = "..."; // no upload

// If you must repaint, minimize the repainted area:
// Use CSS containment to limit paint invalidation scope
element.style.contain = "paint"; // repaints only inside this element
```

---

## 11. GPU-Accelerated CSS Filters

Some CSS `filter` functions run on the GPU as shader passes, others require CPU rasterization:

### GPU-Accelerated Filters (Fast)

```css
/* These run as GPU shader passes — fast, compositor thread */
filter: brightness(0.8); /* multiply RGB channels */
filter: contrast(1.2); /* scale contrast in shader */
filter: saturate(0.5); /* desaturate in shader */
filter: invert(1); /* invert RGB in shader */
filter: hue-rotate(90deg); /* rotate hue in shader */
filter: grayscale(0.5); /* desaturate to gray in shader */
filter: sepia(0.3); /* apply sepia matrix in shader */
filter: opacity(0.8); /* alpha multiply (use opacity property instead) */
```

### CPU-Rasterized Filters (Slow)

```css
/* These require CPU-side rasterization — main thread, expensive */
filter: blur(4px); /* Gaussian blur: O(area × radius²) */
filter: drop-shadow(...); /* shadow calculation: O(area) */
```

### `blur()` Performance Optimization

```css
/* ❌ Animating blur — repaints every frame */
.modal-backdrop:hover {
  filter: blur(20px);
  transition: filter 0.3s;
}

/* ✅ Pre-blur as a separate static element, animate opacity */
.modal-backdrop-blurred {
  /* Pre-rendered blurred version as background-image or CSS trick */
  filter: blur(20px); /* paint ONCE on load */
  opacity: 0;
  transition: opacity 0.3s; /* composite only */
}
.modal-backdrop-blurred.visible {
  opacity: 1;
}

/* Or: Use backdrop-filter instead of filter on the backdrop element */
.modal {
  backdrop-filter: blur(20px);
  /* The browser composites the blur on the GPU using a separate
     "backdrop layer" pass — more efficient than repainting */
}
```

### `backdrop-filter` — GPU Composited Blur

```css
.frosted-glass {
  backdrop-filter: blur(10px) brightness(0.8);
  /* This blurs whatever is BEHIND the element */
  /* Composited on GPU using a backdrop layer — doesn't repaint content behind */
  /* Still GPU-expensive, but more efficient than filter: blur() on the content */
}
```

---

## 12. Checking GPU Usage

### Chrome's GPU Information Page

```
Navigate to: chrome://gpu

Key information:
  Graphics Feature Status:
    "Hardware accelerated" → GPU in use
    "Software only, hardware acceleration unavailable" → GPU disabled

  Important features:
    GPU Compositing: Hardware accelerated ← main compositing
    Rasterization: Hardware accelerated ← GPU rasterization
    WebGL: Hardware accelerated ← WebGL
    Video Decode: Hardware accelerated ← video decode

  GPU Process Memory:
    Shows GPU process memory usage
```

### DevTools GPU Performance

```
Performance tab → GPU track (if enabled)

To enable GPU track:
  Performance panel → ⚙️ (settings) → GPU

Shows:
  GPU memory usage over time
  GPU utilization per frame
  Texture memory in use
```

### System-Level GPU Monitoring

```
Windows: Task Manager → Performance → GPU
  Shows: GPU utilization, GPU memory usage per process

macOS: Activity Monitor → GPU History
  Shows: GPU engine usage

Linux: nvidia-smi (NVIDIA), radeontop (AMD)
  Shows: GPU utilization, VRAM usage

All platforms:
  If browser is consistently using 80%+ GPU: likely compositing too much
  If GPU memory > 500MB for a single page: likely layer explosion
```

---

## 13. GPU Acceleration on Mobile

Mobile GPU performance is fundamentally different from desktop:

### Mobile GPU Architecture

```
Desktop GPU (discrete):
  Dedicated VRAM (GDDR6): fast bandwidth, large capacity
  Dedicated power supply: can draw 150-300W
  Separate silicon from CPU
  Thermal throttling: rare in short bursts

Mobile GPU (integrated):
  Unified memory (LPDDR5): shared with CPU
  Power budget: 2-5W total for chip (GPU gets fraction)
  Same silicon die as CPU (Apple Silicon, Snapdragon)
  Thermal throttling: common after 30-60s sustained load
```

### Mobile-Specific Challenges

```
1. Tile-Based Rendering (TBDR — most mobile GPUs):
   Desktop: Immediate Mode Rendering — processes entire frame at once
   Mobile: Tile-Based Deferred Rendering — splits into 32×32 tiles

   Implication: alpha blending order matters more
   Too many overlapping transparent layers: significant overhead

2. Bandwidth Sensitivity:
   Mobile: 50-200 GB/s memory bandwidth (shared with CPU)
   Desktop: 400-1000 GB/s dedicated GPU bandwidth

   Implication: texture-heavy pages hit mobile bandwidth limits
   Many large textures = mobile-specific slowdowns

3. Thermal Throttling:
   After ~30-60s of heavy GPU work, mobile throttles clock speeds
   60fps for 5 seconds → gradually drops to 30fps → 20fps
   Solution: reduce GPU work in sustained animations
```

### Mobile-Specific Optimizations

```css
/* Reduce layer size on high-DPI mobile */
@media (max-resolution: 2dppx) {
  .heavy-layer {
    will-change: auto;
  } /* don't promote on low-DPI */
}

/* Reduce animation complexity on mobile */
@media (prefers-reduced-motion: reduce) {
  .animated {
    animation: none;
    transition: none;
  }
}
```

```javascript
// Detect mobile and reduce GPU pressure
const isMobile =
  /iPhone|iPad|iPod|Android/i.test(navigator.userAgent) ||
  navigator.hardwareConcurrency <= 4;

if (isMobile) {
  // Disable parallax effects
  // Reduce particle count in WebGL
  // Use simpler shaders
  // Reduce layer count
}
```

---

## 14. Good Practices

### ✅ Use `will-change` only for elements that genuinely animate

```css
/* ✅ Justified: this element always slides in */
.slide-in-menu {
  will-change: transform;
}

/* ✅ Dynamic: promote only when needed */
.card.is-dragging {
  will-change: transform;
}
```

### ✅ Prefer composite-only properties for animations

```css
/* ✅ Pure GPU operations */
.animated {
  transition:
    transform 0.3s,
    opacity 0.3s;
}
```

### ✅ Use `backdrop-filter` instead of `filter` for overlay blur

```css
/* ✅ Backdrop filter — GPU composited, doesn't repaint content */
.overlay {
  backdrop-filter: blur(8px);
}
```

### ✅ Monitor GPU memory in DevTools during development

```
DevTools → Performance → GPU track → check peak GPU memory usage
Keep peak GPU memory under 200MB for mobile-compatible pages
```

### ✅ Use `prefers-reduced-motion` to reduce GPU load for users who prefer it

```css
@media (prefers-reduced-motion: reduce) {
  .parallax {
    transform: none !important;
  }
  .auto-animate {
    animation: none !important;
  }
}
```

---

## 15. Bad Practices

### ❌ Using `transform: translateZ(0)` as a cure-all

```css
/* ❌ Applied everywhere without understanding the cost */
.slow-element {
  transform: translateZ(0);
} /* forces GPU layer */
.flickering {
  transform: translateZ(0);
} /* may or may not help */
.everything * {
  transform: translateZ(0);
} /* layer explosion */
```

### ❌ Ignoring mobile GPU constraints

```javascript
// ❌ Assumes desktop GPU
const PARTICLE_COUNT = 1_000_000;
// Works on desktop, crashes mobile browsers
```

### ❌ Animating `filter: blur()` at high frequency

```css
/* ❌ blur() is one of the most expensive GPU operations */
.search-active .page-content {
  filter: blur(4px);
  transition: filter 0.3s; /* repaints on EVERY animation frame */
}
```

### ❌ Uploading massive textures frequently

```javascript
// ❌ Large canvas that repaints every frame
function animate() {
  ctx.clearRect(0, 0, 4096, 4096);
  drawEverything(); // paints 4096×4096px
  // 4096×4096×4 bytes = 64MB uploaded to GPU per frame
  // At 60fps: 3.84 GB/s bandwidth — exceeds mobile GPU bandwidth
  requestAnimationFrame(animate);
}

// ✅ Only clear and redraw changed areas
function animate() {
  // Dirty rect tracking
  const dirty = getDirtyRegion();
  ctx.clearRect(dirty.x, dirty.y, dirty.w, dirty.h);
  drawRegion(ctx, dirty);
  requestAnimationFrame(animate);
}
```

---

## 16. Common Mistakes

### Mistake 1 — Thinking GPU = always faster

```
GPU is faster than CPU for:
  - Massively parallel pixel operations
  - Matrix transforms on textures
  - Shader computations on large datasets

CPU is faster for:
  - Complex branching logic (JavaScript)
  - Sequential operations
  - Irregular data access patterns
  - Small amounts of data

The GPU pipeline has overhead:
  - Data must be transferred to GPU memory
  - Draw calls have setup cost
  - GPU context switches

For very small operations, CPU is faster.
For large, parallel operations, GPU is faster.
```

### Mistake 2 — `transform` doesn't affect layout

```javascript
// ✅ Confirmed: transform doesn't affect layout
element.style.transform = "translateX(100px)";

// The element's layout position is UNCHANGED:
element.offsetLeft; // same as before transform

// But getBoundingClientRect() DOES reflect the visual position:
element.getBoundingClientRect().left; // 100px more than before

// Siblings, parents — layout completely unaffected by transform
```

### Mistake 3 — `will-change` doesn't take effect immediately

```javascript
// ❌ Adding will-change then immediately animating — no benefit
element.style.willChange = "transform";
element.style.transform = "translateX(100px)"; // layer not created yet!

// ✅ Add will-change, wait for next frame, then animate
element.style.willChange = "transform";
requestAnimationFrame(() => {
  requestAnimationFrame(() => {
    // Layer now created — animation starts with pre-promoted layer
    element.style.transform = "translateX(100px)";
  });
});
```

### Mistake 4 — Not considering 2× DPI in GPU memory estimates

```javascript
// On a 2× DPI display, layer memory is 4× larger than you think:
// 800×400 logical = 1600×800 physical = 5.12MB per layer
// 100 layers = 512MB GPU memory — catastrophic on mobile

// Check devicePixelRatio:
const scale = window.devicePixelRatio; // 2 on Retina, 3 on some phones
const actualPixels = logicalWidth * scale * logicalHeight * scale;
```

---

## 17. Interview-Level Explanation

> **"How does GPU acceleration work in the browser? When does it help and when does it hurt?"**

**Strong answer:**

> "GPU acceleration in browsers works at two levels: GPU compositing, where the GPU merges layer bitmaps into the final screen frame using its vertex and fragment shaders; and GPU rasterization, where Chrome uses the GPU's shader pipeline to convert paint commands into pixel bitmaps faster than the CPU.
>
> The GPU is powerful for this because rendering is inherently parallel — each pixel's color can be computed independently. A modern GPU has thousands of shader cores that can process millions of pixels simultaneously, far faster than a CPU's 8-16 sequential cores.
>
> When an element has its own compositor layer — created by `will-change: transform`, CSS animations on transform/opacity, or `position: fixed` — the GPU can apply transforms and opacity to that layer's texture using matrix math and alpha blending in shaders. This is essentially free in terms of CPU cost, which is why `transform` and `opacity` animations are smooth even when JavaScript is busy.
>
> The key GPU acceleration mechanism: `transform` and `opacity` only change the GPU's rendering parameters (transform matrix, alpha value) for an existing texture. No new pixels are computed — the GPU just uses the existing bitmap at a different position or transparency level.
>
> Where GPU acceleration hurts: creating too many compositor layers exhausts GPU memory. Each layer consumes VRAM proportional to its pixel area — a full-page layer at 1920×1080 is about 8MB. On mobile, which has 1-4GB of shared memory between CPU and GPU, having 100+ layers can cause the browser to start evicting textures, causing checkerboarding and jank.
>
> Additionally, `filter: blur()` is one of the most expensive GPU operations — it requires a Gaussian blur shader pass that scales with both area and blur radius. Animating it triggers repaint on every frame, which includes re-rasterizing and re-uploading the texture to the GPU. The optimization is to use a pre-blurred element and animate its opacity instead."

---

## 18. Exercises

### Exercise 1 — GPU memory budget

Calculate the GPU memory cost of this scenario and determine if it's mobile-safe:

```
- Page has 3× Retina display (devicePixelRatio = 3)
- 200 list items each 400×100 logical pixels with will-change: transform
- 1 hero image: 1200×600 logical pixels on its own layer
- 1 fixed navigation: 1200×60 logical pixels
- Background content layer: 1200×3000 logical pixels
```

<details>
<summary>Solution</summary>

```
Memory formula: width × height × devicePixelRatio² × 4 bytes

List items (200 × 400×100 × 3² × 4):
  = 200 × 400 × 100 × 9 × 4
  = 200 × 1,440,000 bytes
  = 288,000,000 bytes = 288MB ← PROBLEMATIC on mobile

Hero image (1200×600 × 9 × 4):
  = 1200 × 600 × 9 × 4
  = 25,920,000 bytes = 25.9MB

Fixed nav (1200×60 × 9 × 4):
  = 1200 × 60 × 9 × 4
  = 2,592,000 bytes = 2.6MB

Background layer (1200×3000 × 9 × 4):
  = 1200 × 3000 × 9 × 4
  = 129,600,000 bytes = 129.6MB

TOTAL: ~446MB GPU memory

For mobile (budget ~200-300MB total): CATASTROPHIC
Will cause texture evictions, checkerboarding, severe jank

FIX: Remove will-change from list items (save 288MB)
     Apply will-change dynamically only during scroll/animation
     Estimated after fix: ~158MB — acceptable
```

</details>

---

### Exercise 2 — Optimize this animation

```css
/* This animation runs at 15fps on mobile — optimize it */
.notification-banner {
  position: fixed;
  width: 100%;
  height: 60px;
  top: 0;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  box-shadow: 0 4px 20px rgba(102, 126, 234, 0.6);
  transition:
    top 0.3s ease,
    opacity 0.3s ease;
}

.notification-banner.hidden {
  top: -60px;
  opacity: 0;
}
```

<details>
<summary>Solution</summary>

```css
/* Problems:
   1. `top` animation triggers layout every frame
   2. `box-shadow` requires paint every frame
   Both on position:fixed element = forced layout + paint every frame
*/

/* ✅ Optimized: composite-only animation */
.notification-banner {
  position: fixed;
  width: 100%;
  height: 60px;
  top: 0; /* stays at 0 — no layout change */
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  will-change: transform, opacity; /* pre-promote to layer */
  transform: translateY(0); /* use transform for position */
  transition:
    transform 0.3s ease,
    opacity 0.3s ease;
}

.notification-banner.hidden {
  transform: translateY(-60px); /* composite only — no layout */
  opacity: 0; /* composite only — no paint */
}

/* Box shadow: paint it into a ::after pseudo-element */
.notification-banner::after {
  content: "";
  position: absolute;
  inset: 0;
  box-shadow: 0 4px 20px rgba(102, 126, 234, 0.6);
  /* Painted once — no repaint during animation */
  pointer-events: none;
}

/* Result: 
   Before: layout + paint every frame (15fps mobile)
   After: composite only (60fps mobile) */
```

</details>

---

## 🔗 Related Topics

- [`browser-internals/06-composite-layers.md`](./06-composite-layers.md) — Layer creation and management
- [`browser-internals/05-paint-repaint.md`](./05-paint-repaint.md) — What paint produces for the GPU
- [`performance/10-canvas-optimization.md`](../performance/10-canvas-optimization.md) — GPU-optimized canvas patterns
- [`animations/03-compositor-animations.md`](../animations/03-compositor-animations.md) — Compositor-only animation guide
- [`rendering/01-dom-batching.md`](../rendering/01-dom-batching.md) — Reducing GPU upload frequency

---

<div align="center">

**Next:** [`browser-internals/08-critical-rendering-path.md`](./08-critical-rendering-path.md) →

</div>
