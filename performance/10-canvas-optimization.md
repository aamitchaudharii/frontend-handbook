# 10 — Canvas Optimization

> **"Canvas gives you a blank bitmap and a 2D drawing API. The browser makes no promises about performance — that's entirely your job. Every frame is a negotiation between what you want to draw and how little work you can do to draw it."**

The Canvas 2D API is simultaneously one of the most powerful and most misused browser APIs. Used naively, it redraws the entire canvas 60 times per second even when nothing changed. Used well, it renders hundreds of thousands of elements at 60fps by exploiting dirty rectangles, offscreen canvases, layer compositing, and TypedArrays. This document covers every optimization technique, from basic clearing strategies to GPU-accelerated particle systems.

---

## 📚 Table of Contents

1. [The Canvas Rendering Model](#1-the-canvas-rendering-model)
2. [Context State and the Performance Cost](#2-context-state-and-the-performance-cost)
3. [Dirty Rectangle Optimization](#3-dirty-rectangle-optimization)
4. [Offscreen Canvas — Off-Main-Thread Rendering](#4-offscreen-canvas--off-main-thread-rendering)
5. [Layer Compositing with Multiple Canvases](#5-layer-compositing-with-multiple-canvases)
6. [Object Pooling for Canvas Entities](#6-object-pooling-for-canvas-entities)
7. [TypedArrays for Dense Data](#7-typedarrays-for-dense-data)
8. [Pre-rendering and Caching](#8-pre-rendering-and-caching)
9. [Path Optimization](#9-path-optimization)
10. [Text Rendering Optimization](#10-text-rendering-optimization)
11. [Image Rendering Optimization](#11-image-rendering-optimization)
12. [HiDPI and Retina Display Handling](#12-hidpi-and-retina-display-handling)
13. [Profiling Canvas Performance](#13-profiling-canvas-performance)
14. [Good Practices](#14-good-practices)
15. [Bad Practices](#15-bad-practices)
16. [Common Mistakes](#16-common-mistakes)
17. [Interview-Level Explanation](#17-interview-level-explanation)
18. [Exercises](#18-exercises)

---

## 1. The Canvas Rendering Model

Canvas is a retained-mode-free immediate-mode API. There is no scene graph, no automatic redraw — you issue drawing commands and they execute immediately against the pixel buffer.

```
Canvas rendering model:
  1. You call ctx.fillRect(x, y, w, h)
  2. Browser fills those pixels in the canvas bitmap immediately
  3. That's it. No update() later. No retained node.
  4. To "change" something: clear the pixels and redraw

vs DOM rendering model:
  1. You set element.style.left = '100px'
  2. Browser marks element dirty
  3. At next frame: browser computes layout, paints, composites
  4. Automatic — you don't control the pixels
```

### What Canvas Is Good For

```
Canvas excels at:
  ✦ Large numbers of moving entities (1,000+ particles, sprites)
  ✦ Pixel-level manipulation (image filters, effects)
  ✦ Custom data visualizations (charts with 100,000+ data points)
  ✦ Games with many dynamic objects
  ✦ Real-time audio visualizations
  ✦ Drawing applications

Canvas is worse than DOM for:
  ✗ Text-heavy content (accessibility, searchability, selectability)
  ✗ Forms and interactive inputs
  ✗ Content that must be indexed by search engines
  ✗ Simple animations (CSS handles these better)
  ✗ Fewer than ~100 elements (DOM is simpler and often faster)
```

---

## 2. Context State and the Performance Cost

Every canvas draw call reads from the context's current state: fillStyle, strokeStyle, lineWidth, font, transform, etc. State changes are not free — the GPU must be notified of each change.

### Minimizing State Changes

```javascript
// ❌ Alternating state per item — expensive
function drawItems(ctx, items) {
  items.forEach((item) => {
    ctx.fillStyle = item.color; // state change
    ctx.font = item.font; // state change
    ctx.fillRect(item.x, item.y, item.w, item.h);
    ctx.fillText(item.label, item.x, item.y + item.h);
  });
}

// ✅ Batch by state — minimize state changes
function drawItems(ctx, items) {
  // Sort by visual style to group items with same state
  const byColor = new Map();
  items.forEach((item) => {
    if (!byColor.has(item.color)) byColor.set(item.color, []);
    byColor.get(item.color).push(item);
  });

  // Draw all items of each color together — one state change per color
  for (const [color, group] of byColor) {
    ctx.fillStyle = color;
    group.forEach((item) => {
      ctx.fillRect(item.x, item.y, item.w, item.h);
    });
  }
}
```

### save() and restore() Cost

```javascript
// save/restore push/pop the entire context state stack
// Expensive in tight loops

// ❌ save/restore in a loop
particles.forEach((p) => {
  ctx.save(); // push full state stack
  ctx.translate(p.x, p.y);
  ctx.rotate(p.angle);
  ctx.fillStyle = p.color;
  drawParticle(ctx);
  ctx.restore(); // pop full state stack
});

// ✅ Manual state management (or use transform matrix directly)
particles.forEach((p) => {
  // Apply and undo transform manually — no stack push/pop
  ctx.translate(p.x, p.y);
  ctx.rotate(p.angle);

  drawParticle(ctx, p.color);

  ctx.rotate(-p.angle); // undo rotation
  ctx.translate(-p.x, -p.y); // undo translation
});

// ✅✅ setTransform — replace entire matrix at once (best for independent transforms)
particles.forEach((p) => {
  const cos = Math.cos(p.angle);
  const sin = Math.sin(p.angle);
  // setTransform(a, b, c, d, e, f) — complete matrix replacement
  ctx.setTransform(cos, sin, -sin, cos, p.x, p.y);
  ctx.fillStyle = p.color;
  drawParticle(ctx);
});
ctx.setTransform(1, 0, 0, 1, 0, 0); // reset to identity
```

---

## 3. Dirty Rectangle Optimization

The most impactful canvas optimization: don't clear and redraw the entire canvas when only part of it changed.

### Full Canvas Clear (Naive)

```javascript
// ❌ Clear entire canvas every frame — wastes time on unchanged areas
function render(ctx) {
  ctx.clearRect(0, 0, canvas.width, canvas.height); // clear ALL pixels
  entities.forEach((entity) => draw(ctx, entity)); // redraw EVERYTHING
}
```

### Dirty Rectangle Tracking

```javascript
class DirtyRectManager {
  #dirtyRects = [];
  #union = null; // merged bounding box of all dirty areas

  mark(x, y, w, h) {
    // Expand each side by 1px to ensure full coverage (antialiasing)
    this.#dirtyRects.push({
      x: Math.floor(x - 1),
      y: Math.floor(y - 1),
      w: Math.ceil(w + 2),
      h: Math.ceil(h + 2),
    });
    this.#union = null; // invalidate union cache
  }

  get union() {
    if (this.#union) return this.#union;
    if (this.#dirtyRects.length === 0) return null;

    let minX = Infinity,
      minY = Infinity,
      maxX = -Infinity,
      maxY = -Infinity;
    this.#dirtyRects.forEach(({ x, y, w, h }) => {
      minX = Math.min(minX, x);
      minY = Math.min(minY, y);
      maxX = Math.max(maxX, x + w);
      maxY = Math.max(maxY, y + h);
    });
    this.#union = { x: minX, y: minY, w: maxX - minX, h: maxY - minY };
    return this.#union;
  }

  clear() {
    this.#dirtyRects = [];
    this.#union = null;
  }
  get isEmpty() {
    return this.#dirtyRects.length === 0;
  }
}

// Using dirty rects in render loop
const dirty = new DirtyRectManager();

function updateEntity(entity, newX, newY) {
  // Mark old position dirty
  dirty.mark(
    entity.x - entity.radius,
    entity.y - entity.radius,
    entity.radius * 2,
    entity.radius * 2,
  );

  entity.x = newX;
  entity.y = newY;

  // Mark new position dirty
  dirty.mark(
    entity.x - entity.radius,
    entity.y - entity.radius,
    entity.radius * 2,
    entity.radius * 2,
  );
}

function render(ctx) {
  if (dirty.isEmpty) return; // nothing changed — skip frame entirely

  const rect = dirty.union;

  // ✅ Clear only the dirty region
  ctx.clearRect(rect.x, rect.y, rect.w, rect.h);

  // ✅ Clip to dirty region — only redraw what's inside
  ctx.save();
  ctx.beginPath();
  ctx.rect(rect.x, rect.y, rect.w, rect.h);
  ctx.clip();

  // Draw only entities that intersect the dirty region
  entities
    .filter((e) => intersects(e.bounds, rect))
    .forEach((e) => draw(ctx, e));

  ctx.restore();
  dirty.clear();
}
```

---

## 4. Offscreen Canvas — Off-Main-Thread Rendering

`OffscreenCanvas` allows canvas rendering in a Web Worker, completely off the main thread.

### Transferring to a Worker

```javascript
// main.js
const canvas = document.getElementById("main-canvas");
const offscreen = canvas.transferControlToOffscreen();

const worker = new Worker("./render-worker.js", { type: "module" });
worker.postMessage({ canvas: offscreen }, [offscreen]); // transfer ownership

// Main thread can now do other work — rendering is on the worker thread
worker.postMessage({ type: "ADD_PARTICLE", x: 100, y: 200 });
```

```javascript
// render-worker.js
let ctx;
let particles = [];
let lastTime = null;

self.onmessage = ({ data }) => {
  if (data.canvas) {
    ctx = data.canvas.getContext("2d");
    requestAnimationFrame(renderLoop); // rAF available in OffscreenCanvas workers
    return;
  }

  if (data.type === "ADD_PARTICLE") {
    particles.push(createParticle(data.x, data.y));
  }
};

function renderLoop(timestamp) {
  const delta = lastTime ? (timestamp - lastTime) / 1000 : 0;
  lastTime = timestamp;

  // All rendering happens in worker — main thread is completely free
  ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);

  particles = particles.filter((p) => p.life > 0);
  particles.forEach((p) => {
    p.x += p.vx * delta;
    p.y += p.vy * delta;
    p.life -= delta;

    const alpha = Math.max(0, p.life);
    ctx.fillStyle = `rgba(${p.r},${p.g},${p.b},${alpha})`;
    ctx.beginPath();
    ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2);
    ctx.fill();
  });

  requestAnimationFrame(renderLoop);
}
```

### OffscreenCanvas for CPU-Intensive Processing

```javascript
// Process image data in worker without blocking main thread
const worker = new Worker("./image-processor.js");

async function applyFilter(imageData) {
  return new Promise((resolve) => {
    worker.onmessage = ({ data }) => resolve(data.result);
    worker.postMessage({ imageData }, [imageData.data.buffer]); // transfer buffer
  });
}

// render-worker.js
self.onmessage = ({ data: { imageData } }) => {
  // Heavy pixel manipulation on worker thread — main thread unaffected
  const result = applyGaussianBlur(imageData);
  self.postMessage({ result }, [result.data.buffer]);
};
```

---

## 5. Layer Compositing with Multiple Canvases

Stack multiple canvases to separate rendering layers — only redraw changed layers.

```html
<!-- CSS: stack canvases on top of each other -->
<div
  class="canvas-container"
  style="position: relative; width: 800px; height: 600px;"
>
  <!-- Layer 0: background (redraws rarely) -->
  <canvas
    id="background-layer"
    width="800"
    height="600"
    style="position: absolute; top: 0; left: 0;"
  ></canvas>

  <!-- Layer 1: game objects (redraws on each frame) -->
  <canvas
    id="entities-layer"
    width="800"
    height="600"
    style="position: absolute; top: 0; left: 0;"
  ></canvas>

  <!-- Layer 2: UI/HUD (redraws when score changes) -->
  <canvas
    id="ui-layer"
    width="800"
    height="600"
    style="position: absolute; top: 0; left: 0; pointer-events: none;"
  ></canvas>
</div>
```

```javascript
const backgroundCtx = document
  .getElementById("background-layer")
  .getContext("2d");
const entitiesCtx = document.getElementById("entities-layer").getContext("2d");
const uiCtx = document.getElementById("ui-layer").getContext("2d");

// Background: drawn once (or rarely)
function drawBackground() {
  backgroundCtx.clearRect(0, 0, 800, 600);
  // Draw grid, terrain, static elements
  drawGrid(backgroundCtx);
  drawTerrain(backgroundCtx);
}
drawBackground(); // only called when background changes

// Entities: drawn every frame
function drawEntities(delta) {
  entitiesCtx.clearRect(0, 0, 800, 600); // only clear this layer
  entities.forEach((e) => draw(entitiesCtx, e, delta));
}

// UI: drawn when score/health changes
let lastScore = 0;
function drawUI(score, health) {
  if (score === lastScore) return; // no change — skip
  lastScore = score;
  uiCtx.clearRect(0, 0, 800, 600);
  drawScore(uiCtx, score);
  drawHealthBar(uiCtx, health);
}

// Main loop: only updates changed layers
function gameLoop(timestamp) {
  const delta = (timestamp - lastTimestamp) / 1000;

  update(delta); // update game state
  drawEntities(delta); // always redraws (moving entities)
  drawUI(state.score, state.health); // redraws only if score changed

  requestAnimationFrame(gameLoop);
}
```

---

## 6. Object Pooling for Canvas Entities

Creating and destroying JavaScript objects in a canvas render loop triggers garbage collection. Use pooling to eliminate allocations.

```javascript
class ParticlePool {
  constructor(maxSize = 1000) {
    this._pool = [];
    this._active = [];
    this._max = maxSize;

    // Pre-allocate using TypedArrays for cache efficiency
    // Each particle: x, y, vx, vy, r, g, b, life, maxLife, size
    this._data = new Float32Array(maxSize * 10);
    this._free = Array.from({ length: maxSize }, (_, i) => i); // free slot indices
  }

  _slot(index) {
    return index * 10; // offset in _data
  }

  spawn(x, y, options = {}) {
    if (this._free.length === 0) return -1; // pool exhausted

    const index = this._free.pop();
    const s = this._slot(index);

    this._data[s] = x; // x
    this._data[s + 1] = y; // y
    this._data[s + 2] = options.vx ?? (Math.random() - 0.5) * 100;
    this._data[s + 3] = options.vy ?? (Math.random() - 0.5) * 100;
    this._data[s + 4] = options.r ?? 255;
    this._data[s + 5] = options.g ?? 100;
    this._data[s + 6] = options.b ?? 0;
    this._data[s + 7] = options.life ?? 2; // seconds
    this._data[s + 8] = options.life ?? 2; // maxLife
    this._data[s + 9] = options.size ?? 4;

    this._active.push(index);
    return index;
  }

  update(delta) {
    // Update all active particles, collect dead ones
    const dead = [];
    for (const index of this._active) {
      const s = this._slot(index);
      this._data[s] += this._data[s + 2] * delta; // x += vx * dt
      this._data[s + 1] += this._data[s + 3] * delta; // y += vy * dt
      this._data[s + 7] -= delta; // life -= dt

      if (this._data[s + 7] <= 0) dead.push(index);
    }

    // Return dead particles to pool
    for (const index of dead) {
      const i = this._active.indexOf(index);
      if (i !== -1) this._active.splice(i, 1);
      this._free.push(index);
    }
  }

  render(ctx) {
    for (const index of this._active) {
      const s = this._slot(index);
      const alpha = Math.max(0, this._data[s + 7] / this._data[s + 8]);
      const r = this._data[s + 4] | 0;
      const g = this._data[s + 5] | 0;
      const b = this._data[s + 6] | 0;

      ctx.fillStyle = `rgba(${r},${g},${b},${alpha.toFixed(2)})`;
      ctx.beginPath();
      ctx.arc(
        this._data[s],
        this._data[s + 1],
        this._data[s + 9],
        0,
        Math.PI * 2,
      );
      ctx.fill();
    }
  }

  get activeCount() {
    return this._active.length;
  }
}
```

---

## 7. TypedArrays for Dense Data

For large datasets (10,000+ points), TypedArrays are significantly faster than regular arrays for numerical data.

```javascript
// ❌ Regular array: values are boxed, heap-allocated, not cache-friendly
const points = [];
for (let i = 0; i < 100_000; i++) {
  points.push({ x: Math.random() * 800, y: Math.random() * 600, r: 2 });
}
// Each object: ~100 bytes on heap, GC-tracked, pointer chase to read x/y

// ✅ TypedArray: flat, cache-friendly, no GC pressure
// Interleaved layout: [x0, y0, r0, x1, y1, r1, ...]
const STRIDE = 3; // x, y, r per point
const pointsTA = new Float32Array(100_000 * STRIDE);

// Populate
for (let i = 0; i < 100_000; i++) {
  pointsTA[i * STRIDE] = Math.random() * 800;
  pointsTA[i * STRIDE + 1] = Math.random() * 600;
  pointsTA[i * STRIDE + 2] = 2;
}

// Render — tight loop over contiguous memory
function renderPoints(ctx, data, count) {
  for (let i = 0; i < count; i++) {
    const x = data[i * STRIDE];
    const y = data[i * STRIDE + 1];
    const r = data[i * STRIDE + 2];

    ctx.beginPath();
    ctx.arc(x, y, r, 0, Math.PI * 2);
    ctx.fill();
  }
}
```

### Pixel Manipulation with ImageData

```javascript
// Direct pixel access: fastest way to modify large areas
function grayscale(ctx, x, y, width, height) {
  const imageData = ctx.getImageData(x, y, width, height);
  const data = imageData.data; // Uint8ClampedArray: [R,G,B,A, R,G,B,A, ...]

  // TypedArray access — very fast, direct memory
  for (let i = 0; i < data.length; i += 4) {
    const avg = (data[i] + data[i + 1] + data[i + 2]) / 3;
    data[i] = avg; // R
    data[i + 1] = avg; // G
    data[i + 2] = avg; // B
    // data[i + 3] = alpha unchanged
  }

  ctx.putImageData(imageData, x, y);
}

// ✅ SharedArrayBuffer: share pixel data with worker thread without copying
const buffer = new SharedArrayBuffer(width * height * 4); // RGBA
const sharedData = new Uint8ClampedArray(buffer);

// Worker can read/write sharedData while main thread continues
```

---

## 8. Pre-rendering and Caching

Pre-render complex shapes onto offscreen canvases. Use the cached bitmap instead of re-drawing.

```javascript
// Pre-render a complex gradient circle
function createCircleGradient(radius, colorStops) {
  const size = radius * 2;
  const cache = new OffscreenCanvas(size, size);
  const ctx = cache.getContext("2d");

  const gradient = ctx.createRadialGradient(
    radius,
    radius,
    0,
    radius,
    radius,
    radius,
  );
  colorStops.forEach(([stop, color]) => gradient.addColorStop(stop, color));

  ctx.fillStyle = gradient;
  ctx.beginPath();
  ctx.arc(radius, radius, radius, 0, Math.PI * 2);
  ctx.fill();

  return cache; // cached bitmap
}

// Create once
const sunBitmap = createCircleGradient(50, [
  [0, "rgba(255, 255, 150, 1)"],
  [0.5, "rgba(255, 200, 50, 0.8)"],
  [1, "rgba(255, 100, 0, 0)"],
]);

// Draw many times — fast bitmap blit instead of gradient re-computation
function renderSun(ctx, x, y) {
  ctx.drawImage(sunBitmap, x - 50, y - 50); // O(1) blit
}

// Without caching: createRadialGradient + arc + fill on every frame
// With caching: drawImage (GPU texture copy) on every frame
// For 1000 suns: hundreds of ms saved per frame
```

### Sprite Sheet for Animations

```javascript
class SpriteSheet {
  constructor(image, frameWidth, frameHeight) {
    this._image = image;
    this._frameWidth = frameWidth;
    this._frameHeight = frameHeight;
    this._framesPerRow = Math.floor(image.width / frameWidth);
  }

  draw(ctx, frameIndex, x, y, scale = 1) {
    const col = frameIndex % this._framesPerRow;
    const row = Math.floor(frameIndex / this._framesPerRow);

    ctx.drawImage(
      this._image,
      col * this._frameWidth, // source x
      row * this._frameHeight, // source y
      this._frameWidth, // source width
      this._frameHeight, // source height
      x, // dest x
      y, // dest y
      this._frameWidth * scale, // dest width
      this._frameHeight * scale, // dest height
    );
  }
}
```

---

## 9. Path Optimization

### Batch Path Operations

```javascript
// ❌ Separate path per shape — N fill calls
shapes.forEach((shape) => {
  ctx.beginPath();
  ctx.arc(shape.x, shape.y, shape.r, 0, Math.PI * 2);
  ctx.fill(); // GPU draw call per shape
});

// ✅ Single path, single fill — all same-color shapes in one draw call
ctx.fillStyle = "#ff6b35";
ctx.beginPath();
shapes.forEach((shape) => {
  ctx.moveTo(shape.x + shape.r, shape.y);
  ctx.arc(shape.x, shape.y, shape.r, 0, Math.PI * 2);
});
ctx.fill(); // ONE GPU draw call for all shapes
```

### Path2D for Reusable Paths

```javascript
// Create path once, stroke/fill many times
const arrowPath = new Path2D();
arrowPath.moveTo(0, -10);
arrowPath.lineTo(10, 10);
arrowPath.lineTo(-10, 10);
arrowPath.closePath();

// Reuse across frames (path computation already done)
arrows.forEach((arrow) => {
  ctx.save();
  ctx.translate(arrow.x, arrow.y);
  ctx.rotate(arrow.angle);
  ctx.fill(arrowPath); // no path re-computation
  ctx.restore();
});
```

### Simplify Complex Paths

```javascript
// For data visualization: simplify paths to reduce point count
function simplifyPath(points, tolerance = 1) {
  if (points.length <= 2) return points;

  // Ramer-Douglas-Peucker algorithm
  const sqDist = (p1, p2) => {
    const dx = p1[0] - p2[0],
      dy = p1[1] - p2[1];
    return dx * dx + dy * dy;
  };

  const sqSegDist = (p, p1, p2) => {
    let x = p1[0],
      y = p1[1];
    let dx = p2[0] - x,
      dy = p2[1] - y;
    if (dx !== 0 || dy !== 0) {
      const t = ((p[0] - x) * dx + (p[1] - y) * dy) / (dx * dx + dy * dy);
      if (t > 1) {
        x = p2[0];
        y = p2[1];
      } else if (t > 0) {
        x += dx * t;
        y += dy * t;
      }
    }
    return sqDist(p, [x, y]);
  };

  const sqTolerance = tolerance * tolerance;
  const first = 0,
    last = points.length - 1;
  let maxSqDist = 0,
    index = 0;

  for (let i = 1; i < last; i++) {
    const d = sqSegDist(points[i], points[first], points[last]);
    if (d > maxSqDist) {
      maxSqDist = d;
      index = i;
    }
  }

  if (maxSqDist > sqTolerance) {
    const left = simplifyPath(points.slice(0, index + 1), tolerance);
    const right = simplifyPath(points.slice(index), tolerance);
    return [...left.slice(0, -1), ...right];
  }

  return [points[first], points[last]];
}
```

---

## 10. Text Rendering Optimization

Text rendering is expensive — measuring and drawing text has significant overhead.

```javascript
// ❌ measureText on every frame
function drawLabel(ctx, text, x, y) {
  const metrics = ctx.measureText(text); // expensive per call
  ctx.fillText(text, x - metrics.width / 2, y);
}

// ✅ Cache text measurements
const textCache = new Map();

function measureTextCached(ctx, text, font) {
  const key = `${font}:${text}`;
  if (textCache.has(key)) return textCache.get(key);

  ctx.font = font;
  const metrics = ctx.measureText(text);
  const result = { width: metrics.width };
  textCache.set(key, result);
  return result;
}

// ✅ Pre-render text to OffscreenCanvas for frequently-redrawn labels
const labelCache = new Map();

function getTextBitmap(text, font, color) {
  const key = `${text}:${font}:${color}`;
  if (labelCache.has(key)) return labelCache.get(key);

  // Measure
  const tempCtx = new OffscreenCanvas(1, 1).getContext("2d");
  tempCtx.font = font;
  const { width } = tempCtx.measureText(text);

  // Render to bitmap
  const bitmap = new OffscreenCanvas(Math.ceil(width) + 4, 30);
  const bCtx = bitmap.getContext("2d");
  bCtx.font = font;
  bCtx.fillStyle = color;
  bCtx.fillText(text, 2, 20);

  labelCache.set(key, bitmap);
  return bitmap;
}

// Draw many labels with same text: fast bitmap blit
scores.forEach((score) => {
  const label = getTextBitmap(`${score.value} pts`, "14px sans-serif", "white");
  ctx.drawImage(label, score.x, score.y);
});
```

---

## 11. Image Rendering Optimization

### Appropriate Image Sizes

```javascript
// ❌ Drawing a 4000×3000 image scaled to 100×75
ctx.drawImage(largeImage, x, y, 100, 75);
// Browser must scale 12 million pixels to 7,500 pixels on every draw call

// ✅ Downscale once, draw the small version
function scaleImage(image, targetWidth, targetHeight) {
  const canvas = new OffscreenCanvas(targetWidth, targetHeight);
  const ctx = canvas.getContext("2d");
  ctx.drawImage(image, 0, 0, targetWidth, targetHeight);
  return canvas;
}

const thumbnail = scaleImage(largeImage, 100, 75); // done once
// Draw thumbnail (100×75) on every frame — far cheaper
ctx.drawImage(thumbnail, x, y);
```

### createImageBitmap for Fast Decoding

```javascript
// createImageBitmap: decode image on worker thread
// Returns a bitmap ready for GPU upload — no main thread decode penalty

async function loadImage(src) {
  const response = await fetch(src);
  const blob = await response.blob();

  // Decode off the main thread, with optional resize
  const bitmap = await createImageBitmap(blob, {
    resizeWidth: 800,
    resizeHeight: 600,
    resizeQuality: "high",
  });

  return bitmap; // ImageBitmap: ready for drawImage, no decode on use
}

const bitmap = await loadImage("/large-photo.jpg");
ctx.drawImage(bitmap, 0, 0); // instant — already decoded and resized
```

### Avoiding drawImage from Video

```javascript
// ❌ drawImage from video: very expensive (video decode + GPU upload per frame)
function renderWithVideo(ctx, video) {
  ctx.drawImage(video, 0, 0); // decode + upload every frame!
}

// ✅ Capture to ImageBitmap when frame changes, reuse until next frame
let lastBitmap = null;
let lastTime = -1;

async function renderWithVideo(ctx, video) {
  if (video.currentTime !== lastTime) {
    lastBitmap = await createImageBitmap(video); // async decode
    lastTime = video.currentTime;
  }
  if (lastBitmap) ctx.drawImage(lastBitmap, 0, 0); // reuse cached bitmap
}
```

---

## 12. HiDPI and Retina Display Handling

Retina displays have a `devicePixelRatio` (DPR) of 2 or 3 — logical CSS pixels vs physical device pixels.

```javascript
function setupHiDPICanvas(canvas) {
  const dpr = window.devicePixelRatio || 1;

  // Get the CSS (logical) size
  const rect = canvas.getBoundingClientRect();

  // Set actual pixel dimensions
  canvas.width = rect.width * dpr;
  canvas.height = rect.height * dpr;

  // Scale the context to match CSS size
  const ctx = canvas.getContext("2d");
  ctx.scale(dpr, dpr);

  // CSS size remains unchanged:
  canvas.style.width = rect.width + "px";
  canvas.style.height = rect.height + "px";

  return ctx;
  // Now: drawing at CSS coordinates automatically maps to physical pixels
  // ctx.fillRect(0, 0, 100, 100) renders sharp at 200×200 physical pixels
}

// Handle viewport resize
function resizeCanvas(canvas, ctx) {
  const dpr = window.devicePixelRatio || 1;
  const rect = canvas.getBoundingClientRect();

  const physW = Math.round(rect.width * dpr);
  const physH = Math.round(rect.height * dpr);

  if (canvas.width !== physW || canvas.height !== physH) {
    canvas.width = physW;
    canvas.height = physH;
    ctx.scale(dpr, dpr); // re-apply scale after resize
    return true; // canvas was resized — needs full redraw
  }
  return false;
}
```

---

## 13. Profiling Canvas Performance

### Timing Individual Operations

```javascript
// Measure specific canvas operations
function benchmarkCanvasOp(name, fn, iterations = 1000) {
  // Warm up
  for (let i = 0; i < 10; i++) fn();

  const start = performance.now();
  for (let i = 0; i < iterations; i++) fn();
  const elapsed = performance.now() - start;

  console.log(`${name}: ${(elapsed / iterations).toFixed(3)}ms avg`);
}

const canvas = document.createElement("canvas");
canvas.width = canvas.height = 200;
const ctx = canvas.getContext("2d");

benchmarkCanvasOp("arc + fill (individual)", () => {
  ctx.beginPath();
  ctx.arc(100, 100, 50, 0, Math.PI * 2);
  ctx.fill();
});

benchmarkCanvasOp("fillRect", () => {
  ctx.fillRect(50, 50, 100, 100);
});

benchmarkCanvasOp("drawImage (bitmap)", () => {
  ctx.drawImage(prerenderedBitmap, 0, 0);
});
```

### FPS Monitor

```javascript
class CanvasFPSMonitor {
  #frames = [];
  #container;
  #rafId = null;

  constructor(container) {
    this.#container = container;
  }

  start() {
    const loop = (t) => {
      this.#frames.push(t);
      if (this.#frames.length > 60) this.#frames.shift();

      if (this.#frames.length >= 2) {
        const elapsed = this.#frames.at(-1) - this.#frames[0];
        const fps = Math.round((this.#frames.length - 1) / (elapsed / 1000));
        this.#container.textContent = `${fps} FPS`;
      }

      this.#rafId = requestAnimationFrame(loop);
    };
    this.#rafId = requestAnimationFrame(loop);
  }

  stop() {
    if (this.#rafId) cancelAnimationFrame(this.#rafId);
  }
}
```

---

## 14. Good Practices

### ✅ Use `willReadFrequently` for pixel manipulation

```javascript
// ✅ Hint to browser: this canvas will have getImageData called frequently
// Browser may use CPU rasterization instead of GPU (faster for read-back)
const ctx = canvas.getContext("2d", { willReadFrequently: true });

// Then calling getImageData is faster (no GPU→CPU transfer needed)
const imageData = ctx.getImageData(0, 0, width, height);
```

### ✅ Use integer coordinates when possible

```javascript
// ❌ Sub-pixel coordinates force anti-aliasing computation
ctx.fillRect(10.5, 20.7, 100.3, 50.8);

// ✅ Integer coords: crisp pixels, no anti-aliasing overhead
ctx.fillRect(10, 20, 100, 50);
ctx.fillRect(Math.round(x), Math.round(y), w, h);
```

### ✅ Avoid globalAlpha in loops

```javascript
// ❌ Setting globalAlpha per item — state change per item
items.forEach((item) => {
  ctx.globalAlpha = item.alpha;
  ctx.fillRect(item.x, item.y, item.w, item.h);
});

// ✅ Encode alpha in fillStyle using rgba()
items.forEach((item) => {
  ctx.fillStyle = `rgba(${item.r}, ${item.g}, ${item.b}, ${item.alpha})`;
  ctx.fillRect(item.x, item.y, item.w, item.h);
});

// ✅✅ Sort by alpha + color and batch
```

---

## 15. Bad Practices

### ❌ Reading pixels immediately after drawing (GPU stall)

```javascript
// ❌ Forces GPU→CPU synchronization — very slow
ctx.fillRect(0, 0, 100, 100);
const data = ctx.getImageData(0, 0, 100, 100); // stalls until GPU finishes

// ✅ If reading is required, batch reads away from writes
ctx.fillRect(0, 0, 100, 100);
ctx.drawImage(image, 100, 0);
// ... more drawing ...
requestAnimationFrame(() => {
  // Read in next frame — GPU has finished by then
  const data = ctx.getImageData(0, 0, 100, 100);
});
```

### ❌ Creating gradients or patterns every frame

```javascript
// ❌ Creating a gradient on every render call
function drawSky(ctx) {
  const gradient = ctx.createLinearGradient(0, 0, 0, 600); // NEW object per frame
  gradient.addColorStop(0, "#001f3f");
  gradient.addColorStop(1, "#87CEEB");
  ctx.fillStyle = gradient;
  ctx.fillRect(0, 0, 800, 600);
}

// ✅ Create once, reuse
const skyGradient = ctx.createLinearGradient(0, 0, 0, 600);
skyGradient.addColorStop(0, "#001f3f");
skyGradient.addColorStop(1, "#87CEEB");

function drawSky(ctx) {
  ctx.fillStyle = skyGradient; // reuse same gradient
  ctx.fillRect(0, 0, 800, 600);
}
```

---

## 16. Common Mistakes

### Mistake 1 — Not accounting for devicePixelRatio

```javascript
// ❌ Blurry on Retina displays
canvas.width = 800; // CSS pixels
canvas.height = 600;
// On DPR=2: renders 800×600 pixels into a 1600×1200 physical area → blurry

// ✅ See Section 12 — multiply by devicePixelRatio
canvas.width = 800 * devicePixelRatio;
canvas.height = 600 * devicePixelRatio;
ctx.scale(devicePixelRatio, devicePixelRatio);
```

### Mistake 2 — Using CSS to resize instead of canvas attributes

```javascript
// ❌ Canvas rendered at 300×150 (default), stretched to 800×600 by CSS — blurry
canvas.style.width = "800px";
canvas.style.height = "600px";
// canvas.width is still 300, canvas.height still 150

// ✅ Set canvas attributes for render resolution
canvas.width = 800;
canvas.height = 600;
// CSS size can then match or be set separately for responsive layout
```

### Mistake 3 — Calling ctx.arc for every circle individually

```javascript
// ❌ 10,000 individual GPU draw calls
for (let i = 0; i < 10_000; i++) {
  ctx.beginPath();
  ctx.arc(points[i].x, points[i].y, 3, 0, Math.PI * 2);
  ctx.fill(); // GPU call per circle
}

// ✅ Batch all same-color circles into one fill call
ctx.beginPath();
for (let i = 0; i < 10_000; i++) {
  ctx.moveTo(points[i].x + 3, points[i].y);
  ctx.arc(points[i].x, points[i].y, 3, 0, Math.PI * 2);
}
ctx.fill(); // ONE GPU call for all 10,000 circles
```

---

## 17. Interview-Level Explanation

> **"How do you optimize a canvas application rendering thousands of elements?"**

**Strong answer:**

> "Canvas optimization comes down to three things: minimize how much you redraw, minimize how many GPU calls you make, and minimize GC pressure in the render loop.
>
> The biggest win is usually dirty rectangle tracking — only clearing and redrawing the pixels that actually changed. If 10 out of 1,000 entities moved, you mark their old and new positions dirty, compute the union bounding box, clip to that area, and only redraw entities that intersect it. The other 990 entities are untouched.
>
> For GPU draw calls, batching is critical. Calling `ctx.fill()` is a GPU call. If you call it once per entity, you get N GPU calls per frame. Instead, group entities by their visual state — all entities of the same color — build one composite path with multiple `arc()` calls, then call `fill()` once. For 10,000 circles of the same color, that's the difference between 10,000 GPU calls and 1.
>
> For complex reused shapes, pre-rendering to an OffscreenCanvas is extremely effective. Compute a gradient circle once, cache it as an OffscreenCanvas bitmap. On every subsequent frame, `drawImage()` blits the bitmap directly — much faster than re-computing a gradient.
>
> To eliminate GC pressure, use TypedArrays for particle data instead of object arrays. Position, velocity, color, life — all stored in a Float32Array as interleaved data. The render loop reads contiguous memory with no pointer chasing, no GC involvement. For particles specifically, an object pool preallocates all the TypedArray storage upfront.
>
> For truly heavy computation — image processing, particle physics — OffscreenCanvas allows the entire render loop to run in a Web Worker. The main thread is completely free, and even a GC pause or long task on the main thread can't drop animation frames."

---

## 18. Exercises

### Exercise 1 — Optimize this particle system

```javascript
// ❌ This particle system drops below 30fps at 1000 particles. Why, and how to fix it?

function createParticle(x, y) {
  return {
    x,
    y,
    vx: (Math.random() - 0.5) * 100,
    vy: (Math.random() - 0.5) * 100,
    color: `hsl(${Math.random() * 360},100%,50%)`,
    life: 1 + Math.random(),
  };
}

let particles = [];

function render(ctx, delta) {
  ctx.clearRect(0, 0, 800, 600);

  particles = particles.filter((p) => p.life > 0);

  particles.forEach((p) => {
    p.x += p.vx * delta;
    p.y += p.vy * delta;
    p.life -= delta;

    ctx.globalAlpha = Math.max(0, p.life);
    ctx.fillStyle = p.color;
    ctx.beginPath();
    ctx.arc(p.x, p.y, 4, 0, Math.PI * 2);
    ctx.fill(); // ONE fill per particle
  });
  ctx.globalAlpha = 1;
}
```

<details>
<summary>Solution</summary>

```
Performance problems:
1. `particles.filter()` creates a new array every frame → GC pressure
2. Dynamic HSL string color per particle → string allocation per frame
3. ctx.fill() called per particle → 1000 GPU draw calls per frame
4. ctx.globalAlpha set per particle → 1000 state changes
5. ctx.fillStyle set per particle → 1000 state changes

Optimizations:
1. Use TypedArray for particle data (no per-particle GC allocation)
2. Sort particles by color bucket, batch by alpha range
3. One ctx.fill() per color group instead of per particle
4. Precompute colors as RGB values, build rgba() strings once
5. Use pool instead of filter/array recreation
```

```javascript
// Optimized version
const STRIDE = 7; // x, y, vx, vy, hue, life, maxLife
const MAX = 2000;
const data = new Float32Array(MAX * STRIDE);
let active = 0;
const freeList = Array.from({ length: MAX }, (_, i) => MAX - 1 - i);

function spawnParticle(x, y) {
  if (freeList.length === 0) return;
  const idx = freeList.pop() * STRIDE;
  data[idx] = x;
  data[idx + 1] = y;
  data[idx + 2] = (Math.random() - 0.5) * 100;
  data[idx + 3] = (Math.random() - 0.5) * 100;
  data[idx + 4] = Math.random() * 360;
  data[idx + 5] = 1 + Math.random(); // life
  data[idx + 6] = data[idx + 5]; // maxLife
  active++;
}

function render(ctx, delta) {
  ctx.clearRect(0, 0, 800, 600);

  // Build alpha buckets for batch rendering
  const buckets = new Map(); // "hue:alphaBucket" → indices
  const dead = [];

  for (let i = 0; i < MAX; i++) {
    const idx = i * STRIDE;
    if (data[idx + 5] <= 0) continue;

    data[idx] += data[idx + 2] * delta;
    data[idx + 1] += data[idx + 3] * delta;
    data[idx + 5] -= delta;

    if (data[idx + 5] <= 0) {
      dead.push(i);
      continue;
    }

    const hue = Math.round(data[idx + 4] / 30) * 30; // bucket hue to 30° increments
    const alphaBkt = Math.floor((data[idx + 5] / data[idx + 6]) * 5) / 5;
    const key = `${hue}:${alphaBkt}`;

    if (!buckets.has(key))
      buckets.set(key, { hue, alpha: alphaBkt, indices: [] });
    buckets.get(key).indices.push(i);
  }

  // One fill per bucket
  for (const { hue, alpha, indices } of buckets.values()) {
    ctx.fillStyle = `hsla(${hue},100%,50%,${alpha})`;
    ctx.beginPath();
    for (const i of indices) {
      const x = data[i * STRIDE],
        y = data[i * STRIDE + 1];
      ctx.moveTo(x + 4, y);
      ctx.arc(x, y, 4, 0, Math.PI * 2);
    }
    ctx.fill();
  }

  // Release dead particles
  dead.forEach((i) => {
    freeList.push(i);
    active--;
  });
}
```

</details>

---

## 🔗 Related Topics

- [`browser-internals/07-gpu-acceleration.md`](../browser-internals/07-gpu-acceleration.md) — GPU rendering and texture uploads
- [`javascript-core/12-web-workers.md`](../javascript-core/12-web-workers.md) — OffscreenCanvas with Web Workers
- [`javascript-core/09-garbage-collection.md`](../javascript-core/09-garbage-collection.md) — Object pooling for GC reduction
- [`performance/04-raf-optimization.md`](./04-raf-optimization.md) — rAF for canvas animation loops

---

<div align="center">

**Next:** [`performance/11-svg-optimization.md`](./11-svg-optimization.md) →

</div>
