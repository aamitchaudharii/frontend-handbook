# 28 — Typed Arrays and Binary Data

> **"Most JavaScript developers never need to think about raw bytes. But when you do — parsing a binary file format, implementing a network protocol, processing audio or image data, sharing memory between workers — JavaScript's binary data APIs are surprisingly capable. Typed arrays are the bridge between JavaScript's high-level world and the raw, flat memory that the platform actually runs on."**

🔴 **Level: Senior**

---

## 📚 Table of Contents

1. [Why Binary Data?](#1-why-binary-data)
2. [ArrayBuffer — Raw Memory](#2-arraybuffer--raw-memory)
3. [TypedArray Views](#3-typedarray-views)
4. [DataView — Precise Byte Control](#4-dataview--precise-byte-control)
5. [Endianness](#5-endianness)
6. [Typed Arrays as Regular Arrays](#6-typed-arrays-as-regular-arrays)
7. [SharedArrayBuffer and Atomics](#7-sharedarraybuffer-and-atomics)
8. [Practical: Parsing a Binary Protocol](#8-practical-parsing-a-binary-protocol)
9. [Practical: Working with Images (ImageData)](#9-practical-working-with-images-imagedata)
10. [Practical: Audio Processing (Web Audio)](#10-practical-audio-processing-web-audio)
11. [Blob and File](#11-blob-and-file)
12. [Performance Notes](#12-performance-notes)
13. [Common Mistakes](#13-common-mistakes)
14. [Exercises](#14-exercises)

---

## 1. Why Binary Data?

```javascript
// Standard JavaScript string/number representations are high-level.
// For certain tasks, you need to work with RAW BYTES:

// 1. Network protocols (WebSocket binary frames, custom TCP-over-WS)
// 2. File parsing (PNG, JPEG, PDF, ZIP, WAV header parsing)
// 3. Web Audio API (PCM audio samples are Float32Array)
// 4. Canvas pixel manipulation (RGBA as Uint8ClampedArray)
// 5. WebGL (vertex buffers, texture data)
// 6. WebAssembly (share memory with WASM via SharedArrayBuffer)
// 7. Crypto (hashing, encryption — raw byte input/output)
// 8. Performance-critical data: TypedArrays are faster than regular arrays
//    for numeric computations because every element is the same known size

// THE MODEL:
// ArrayBuffer — the raw memory (just bytes, no interpretation)
//      ↓
// View (TypedArray or DataView) — interprets those bytes as numbers

// Think of ArrayBuffer as the film negative, and a TypedArray as a
// specific print from that negative — a different print of the same
// negative gives you a different interpretation of the same bytes.
```

---

## 2. ArrayBuffer — Raw Memory

```javascript
// ArrayBuffer: a fixed-length block of raw binary data
const buffer = new ArrayBuffer(16); // 16 BYTES of zeroed memory

buffer.byteLength; // 16

// You CANNOT read or write an ArrayBuffer directly — you need a View
const raw = buffer[0]; // undefined — no direct access!

// Slicing: creates a COPY of a range of bytes
const slice = buffer.slice(0, 8); // new ArrayBuffer, bytes 0-7 copied
buffer.byteLength; // still 16 — slice() doesn't modify the original

// Transferring (zero-copy move — ES2023)
const { buffer: transferred } = buffer.transfer();
// The original `buffer` is now detached (byteLength = 0), `transferred` owns the memory
// Used for efficiently passing buffers to workers without copying
```

---

## 3. TypedArray Views

```javascript
// TypedArrays interpret an ArrayBuffer's bytes as a specific numeric type
// All TypedArrays share the same API, just with different element sizes

const buffer = new ArrayBuffer(16);

// The 11 TypedArray types:
//  Type                 Bytes/element  Range
//  Int8Array                1          -128 to 127
//  Uint8Array               1          0 to 255
//  Uint8ClampedArray        1          0 to 255 (clamped, not wrap-around)
//  Int16Array               2          -32768 to 32767
//  Uint16Array              2          0 to 65535
//  Int32Array               4          -2^31 to 2^31-1
//  Uint32Array              4          0 to 2^32-1
//  Float32Array             4          ~7 decimal digits precision
//  Float64Array             8          ~15 decimal digits precision (= JS number)
//  BigInt64Array            8          -2^63 to 2^63-1 (BigInt values)
//  BigUint64Array           8          0 to 2^64-1

// Create from an ArrayBuffer:
const int32 = new Int32Array(buffer); // 4 elements (16 bytes / 4 bytes each)
const uint8 = new Uint8Array(buffer); // 16 elements (16 bytes / 1 byte each)

// They SHARE the same underlying buffer — a write through one is visible in the other!
int32[0] = 0x01020304; // write 4 bytes as a 32-bit integer
uint8[0]; // first byte of that integer (value depends on endianness)

// Create with its OWN new buffer (most common usage):
const data = new Float32Array(1024); // 1024 floats = 4096 bytes, all 0.0
data[0] = 3.14;
data.buffer.byteLength; // 4096

// Create from an array (copies data into a new buffer):
const typed = new Int32Array([1, 2, 3, 4, 5]);
typed.length; // 5
typed.byteLength; // 20 (5 × 4 bytes)
typed.BYTES_PER_ELEMENT; // 4

// Create a view into a sub-range of a buffer (byteOffset, length):
const buffer2 = new ArrayBuffer(32);
const view = new Int32Array(buffer2, 8, 4); // start at byte 8, 4 elements
view.byteOffset; // 8
view.length; // 4
view.byteLength; // 16

// Uint8ClampedArray: values outside [0, 255] are clamped, not wrapped
const clamped = new Uint8ClampedArray([300, -5, 128]);
// [255, 0, 128] — 300 clamped to 255, -5 clamped to 0
// (Uint8Array would wrap: 300 % 256 = 44, (-5 + 256) = 251)
```

---

## 4. DataView — Precise Byte Control

```javascript
// DataView: read/write specific numeric types at specific byte offsets
// Gives full control over byte order (endianness) — essential for protocols

const buffer = new ArrayBuffer(32);
const view = new DataView(buffer);

// Writing (offset in bytes, value, [littleEndian = false])
view.setUint8(0, 0xff); // 1 byte at offset 0
view.setUint16(1, 0x0102); // 2 bytes at offset 1 (big-endian by default)
view.setUint32(3, 0xdeadbeef); // 4 bytes at offset 3
view.setFloat32(7, 3.14); // 4 bytes at offset 7
view.setFloat64(11, 2.718281828); // 8 bytes at offset 11
view.setInt32(19, -42, true); // 4 bytes at offset 19, LITTLE-ENDIAN

// Reading (offset, [littleEndian = false])
view.getUint8(0); // 255
view.getUint16(1); // 0x0102 = 258
view.getUint32(3); // 0xDEADBEEF = 3735928559
view.getFloat32(7); // ~3.14 (float32 precision)
view.getFloat64(11); // 2.718281828
view.getInt32(19, true); // -42 (reading back as little-endian)

// DataView vs TypedArray:
// TypedArray: fast, convenient, but ALL elements are the same type
// DataView:   slower, but can read ANY type at ANY byte offset
//             Essential for parsing mixed-type binary formats (file headers, network packets)
```

---

## 5. Endianness

```javascript
// Endianness: the byte ORDER in which multi-byte numbers are stored
//
// BIG-ENDIAN (network byte order): most significant byte FIRST
//   0x12345678 stored as: 12 34 56 78
//
// LITTLE-ENDIAN (x86, ARM default): least significant byte FIRST
//   0x12345678 stored as: 78 56 34 12
//
// Most network protocols use BIG-ENDIAN (DataView default)
// Most CPUs (x86, ARM) are LITTLE-ENDIAN

// Detecting the host's endianness:
function isLittleEndian() {
  const buffer = new ArrayBuffer(2);
  new Uint16Array(buffer)[0] = 0x0102;
  return new Uint8Array(buffer)[0] === 0x02; // little-endian if LSB is first
}

// TypedArrays always use the NATIVE (host) byte order — can't specify endianness
// DataView lets you specify explicitly on every read/write — use for protocols

// Example: reading a PNG signature (big-endian 32-bit numbers)
async function parsePngSignature(arrayBuffer) {
  const view = new DataView(arrayBuffer);
  // PNG signature: 8 bytes: 89 50 4E 47 0D 0A 1A 0A
  const sig = Array.from(new Uint8Array(arrayBuffer, 0, 8))
    .map((b) => b.toString(16).padStart(2, "0"))
    .join(" ");
  return sig === "89 50 4e 47 0d 0a 1a 0a"; // is it a valid PNG?
}
```

---

## 6. Typed Arrays as Regular Arrays

```javascript
// TypedArrays support most Array methods — but return TypedArrays, not plain arrays
const ta = new Float32Array([1, 2, 3, 4, 5]);

ta.map((x) => x * 2); // Float32Array [2, 4, 6, 8, 10]
ta.filter((x) => x > 2); // Float32Array [3, 4, 5]
ta.reduce((a, b) => a + b, 0); // 15
ta.slice(1, 3); // Float32Array [2, 3] (same buffer? no — slice copies)
ta.subarray(1, 3); // Float32Array [2, 3] (SHARES the buffer — no copy!)

// Iteration
for (const val of ta) {
  /* ... */
}
[...ta]; // [1, 2, 3, 4, 5] — spread to regular array

// NOT available on TypedArrays:
// push(), pop(), shift(), unshift(), splice(), concat()
// (TypedArrays have FIXED length — can't change size after creation)

// set(): copy values from array into TypedArray (fast bulk write)
const dest = new Uint8Array(8);
dest.set([1, 2, 3, 4], 0); // copy at offset 0
dest.set([5, 6, 7, 8], 4); // copy at offset 4
// dest = [1, 2, 3, 4, 5, 6, 7, 8]

// Efficient copy between TypedArrays of the same type:
const src = new Uint8Array([10, 20, 30]);
const copy = new Uint8Array(src); // creates a new buffer and copies
// OR:
const dest2 = new Uint8Array(src.length);
dest2.set(src); // copy without creating an intermediate array
```

---

## 7. SharedArrayBuffer and Atomics

```javascript
// SharedArrayBuffer: an ArrayBuffer that can be SHARED between the main thread
// and workers (and between workers) — no copying needed!
// (Requires COOP/COEP headers in browser: Cross-Origin-Opener-Policy,
//  Cross-Origin-Embedder-Policy — enabled for security against Spectre)

const shared = new SharedArrayBuffer(4 * 4); // 4 int32s

// Main thread:
const mainView = new Int32Array(shared);
mainView[0] = 100;

// Worker thread:
// const workerView = new Int32Array(shared); // same memory!
// workerView[0]; // 100 — sees main thread's write immediately

// PROBLEM: concurrent access without synchronization → RACE CONDITION
// Two threads writing to the same index simultaneously = undefined behavior

// SOLUTION: Atomics — guaranteed atomic (indivisible) operations
Atomics.store(mainView, 0, 42); // atomically write 42 at index 0
Atomics.load(mainView, 0); // atomically read index 0 → 42
Atomics.add(mainView, 0, 1); // atomically increment by 1
Atomics.compareExchange(mainView, 0, 42, 99); // if current === 42, write 99
Atomics.exchange(mainView, 0, 77); // atomically write, return previous value

// Waiting and waking (semaphore-like synchronization):
// Worker:
Atomics.wait(mainView, 0, 0); // block until mainView[0] !== 0
// (only in workers — not main thread)

// Main thread:
Atomics.notify(mainView, 0, 1); // wake 1 waiting thread at index 0

// Use case: implementing a ring buffer or message queue between worker and main thread
// without copying data back and forth via postMessage
```

---

## 8. Practical: Parsing a Binary Protocol

```javascript
// Parse a simple binary message format:
// [1 byte: message type][2 bytes: payload length][N bytes: payload]
function parseMessage(buffer) {
  const view = new DataView(buffer);
  const type = view.getUint8(0);
  const payloadLength = view.getUint16(1, false); // big-endian
  const payload = new Uint8Array(buffer, 3, payloadLength);
  return { type, payload };
}

// Build a binary message:
function buildMessage(type, payloadBytes) {
  const buffer = new ArrayBuffer(3 + payloadBytes.length);
  const view = new DataView(buffer);
  view.setUint8(0, type);
  view.setUint16(1, payloadBytes.length, false); // big-endian
  new Uint8Array(buffer).set(payloadBytes, 3);
  return buffer;
}

// String ↔ bytes conversion:
const encoder = new TextEncoder();
const decoder = new TextDecoder();

const bytes = encoder.encode("Hello, World!"); // Uint8Array of UTF-8 bytes
const text = decoder.decode(bytes); // 'Hello, World!'

// Build a "hello" message:
const payload = encoder.encode("Hello");
const msg = buildMessage(0x01, payload);
// → [0x01, 0x00, 0x05, 0x48, 0x65, 0x6C, 0x6C, 0x6F]
//    type   len(5)       H     e     l     l     o
```

---

## 9. Practical: Working with Images (ImageData)

```javascript
// Canvas ImageData uses Uint8ClampedArray: [R, G, B, A, R, G, B, A, ...]
const canvas = document.createElement("canvas");
canvas.width = 100;
canvas.height = 100;
const ctx = canvas.getContext("2d");

// Draw something first
ctx.fillStyle = "red";
ctx.fillRect(0, 0, 100, 100);

// Get pixel data as a flat Uint8ClampedArray
const imageData = ctx.getImageData(0, 0, 100, 100);
const pixels = imageData.data; // Uint8ClampedArray, length = 100 * 100 * 4

// Read pixel at (x, y):
function getPixel(pixels, width, x, y) {
  const offset = (y * width + x) * 4;
  return {
    r: pixels[offset],
    g: pixels[offset + 1],
    b: pixels[offset + 2],
    a: pixels[offset + 3],
  };
}

// Convert to grayscale in place (fast loop — avoid creating objects per pixel)
for (let i = 0; i < pixels.length; i += 4) {
  const gray =
    0.2126 * pixels[i] + 0.7152 * pixels[i + 1] + 0.0722 * pixels[i + 2];
  pixels[i] = pixels[i + 1] = pixels[i + 2] = gray; // R, G, B = gray
  // pixels[i + 3] = alpha, leave unchanged
}

// Write modified pixels back to canvas
ctx.putImageData(imageData, 0, 0);

// For performance: use Int32Array on the same buffer to write all 4 bytes
// of a pixel atomically (one 32-bit write instead of four 8-bit writes)
const int32View = new Int32Array(pixels.buffer);
// RGBA little-endian: 0xAABBGGRR
int32View[pixelIndex] = (255 << 24) | (0 << 16) | (0 << 8) | 255; // pure red, full opacity
```

---

## 10. Practical: Audio Processing (Web Audio)

```javascript
// Web Audio API uses Float32Array for audio samples
// Each sample is a 32-bit float in range [-1.0, 1.0]

const audioCtx = new AudioContext();

// Generate a sine wave (440 Hz) as a Float32Array
function generateSine(frequency, duration, sampleRate = audioCtx.sampleRate) {
  const samples = new Float32Array(Math.floor(duration * sampleRate));
  const omega = (2 * Math.PI * frequency) / sampleRate;

  for (let i = 0; i < samples.length; i++) {
    samples[i] = Math.sin(omega * i);
  }
  return samples;
}

// Create an AudioBuffer and put the samples in it
async function playSine(frequency, duration) {
  const sampleRate = audioCtx.sampleRate;
  const samples = generateSine(frequency, duration, sampleRate);
  const audioBuffer = audioCtx.createBuffer(1, samples.length, sampleRate);

  // copyToChannel uses Float32Array internally — no data copying overhead
  audioBuffer.copyToChannel(samples, 0);

  const source = audioCtx.createBufferSource();
  source.buffer = audioBuffer;
  source.connect(audioCtx.destination);
  source.start();
}

// AudioWorkletProcessor: runs on the audio thread, receives Float32Array chunks
class GainProcessor extends AudioWorkletProcessor {
  process(inputs, outputs, parameters) {
    const input = inputs[0][0]; // Float32Array of 128 samples
    const output = outputs[0][0]; // Float32Array to fill

    if (input) {
      for (let i = 0; i < input.length; i++) {
        output[i] = input[i] * 0.5; // 50% gain reduction
      }
    }
    return true; // keep processor alive
  }
}
```

---

## 11. Blob and File

```javascript
// Blob: immutable raw binary data with a MIME type
// (higher level than ArrayBuffer — works with the browser's fetch/file APIs)

const blob = new Blob(["Hello, World!"], { type: "text/plain" });
blob.size; // 13
blob.type; // 'text/plain'

// Create from ArrayBuffer:
const buffer = new ArrayBuffer(8);
const blob2 = new Blob([buffer]);

// File is a Blob subclass (has a name and lastModified date)
// You receive File objects from <input type="file"> and drag-and-drop

// Convert between Blob and ArrayBuffer:
const asArrayBuffer = await blob.arrayBuffer(); // Blob → ArrayBuffer
const asBlob = new Blob([asArrayBuffer]); // ArrayBuffer → Blob
const asText = await blob.text(); // Blob → string

// Object URL: give a Blob a temporary URL (for <img src>, <a href>, etc.)
const url = URL.createObjectURL(blob);
imageEl.src = url;
// ⚠️ ALWAYS revoke when done — prevents memory leak
URL.revokeObjectURL(url);

// Reading a File from an <input>
inputEl.addEventListener("change", async () => {
  const file = inputEl.files[0]; // File object (extends Blob)

  // As text
  const text = await file.text();

  // As ArrayBuffer (for binary processing)
  const buffer = await file.arrayBuffer();
  const view = new DataView(buffer);

  // As a stream (for very large files — avoids loading all into memory)
  const reader = file.stream().getReader();
});
```

---

## 12. Performance Notes

```javascript
// 1. TypedArrays have PREDICTABLE element types → JIT compilers can
//    optimize loops over them far better than regular arrays
//    (which can hold mixed types, preventing many optimizations)
//
//    Regular array of numbers: ~0.5-2ns per element access
//    Float32Array:             ~0.1-0.5ns per element access
//    (rough estimates; varies greatly by engine and context)

// 2. set() is a bulk copy — use it instead of element-by-element assignment
//    when copying large amounts of data between TypedArrays

// 3. subarray() creates a view WITHOUT copying — use it instead of slice()
//    when you don't need an independent copy:
const sub = typedArray.subarray(10, 20); // same buffer, no allocation

// 4. For WebGL and WebAudio, passing TypedArrays DIRECTLY avoids conversion
//    gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(vertices), gl.STATIC_DRAW);
//    // NOT:  gl.bufferData(gl.ARRAY_BUFFER, vertices, gl.STATIC_DRAW);
//    // (plain arrays must be converted internally)

// 5. Allocate TypedArrays ONCE and reuse — allocation + GC pressure matters
//    in hot paths (animation loops, audio callbacks):
const scratchBuffer = new Float32Array(1024); // allocated once
function processFrame() {
  // reuse scratchBuffer — no allocation in this hot path
}
```

---

## 13. Common Mistakes

### Mistake 1 — Out-of-bounds write is silent

```javascript
// TypedArray writes out of bounds are SILENTLY IGNORED (no throw, no error)
const ta = new Uint8Array(4);
ta[10] = 255; // silently ignored — no error, ta unchanged
ta[10]; // undefined (not 255!)

// ✅ Always validate indices when writing from untrusted input
function safeWrite(ta, index, value) {
  if (index < 0 || index >= ta.length)
    throw new RangeError(`Index ${index} out of bounds`);
  ta[index] = value;
}
```

### Mistake 2 — Detached buffer access throws

```javascript
const buffer = new ArrayBuffer(8);
const view = new Int32Array(buffer);

const { buffer: transferred } = buffer.transfer();
// buffer is now DETACHED

view[0]; // ❌ TypeError: Cannot perform 'Get' on a detached ArrayBuffer
// ✅ Only read/write TypedArray views before transferring the underlying buffer
```

### Mistake 3 — Uint8Array vs Uint8ClampedArray for image data

```javascript
// ❌ Using Uint8Array for canvas pixel manipulation
// won't automatically clamp values — 256 wraps to 0, -1 wraps to 255
const pixels = new Uint8Array(imageData.data.buffer);
pixels[0] = 300; // wraps to 44 (300 % 256), not the expected 255

// ✅ Canvas ImageData.data is Uint8ClampedArray for a reason — use it directly
const pixels2 = imageData.data; // Uint8ClampedArray
pixels2[0] = 300; // clamped to 255 ✅
```

---

## 14. Exercises

### Exercise 1 — Count bytes matching a value

```javascript
// Write a function that takes an ArrayBuffer and a byte value (0-255),
// and returns how many times that byte appears.
// Use TypedArrays — no converting to a regular array.
function countByte(buffer, byteValue) {
  /* ... */
}
```

<details>
<summary>Solution</summary>

```javascript
function countByte(buffer, byteValue) {
  const view = new Uint8Array(buffer);
  let count = 0;
  for (let i = 0; i < view.length; i++) {
    if (view[i] === byteValue) count++;
  }
  return count;
}
// Or with reduce (slightly slower but idiomatic):
function countByte(buffer, byteValue) {
  return new Uint8Array(buffer).reduce(
    (n, b) => n + (b === byteValue ? 1 : 0),
    0,
  );
}
```

</details>

### Exercise 2 — Encode a little-endian 32-bit integer

```javascript
// Write a function int32ToBytes(n) that returns a Uint8Array of 4 bytes
// representing n as a little-endian 32-bit signed integer.
// int32ToBytes(1) → [1, 0, 0, 0]
// int32ToBytes(256) → [0, 1, 0, 0]
```

<details>
<summary>Solution</summary>

```javascript
function int32ToBytes(n) {
  const buffer = new ArrayBuffer(4);
  new DataView(buffer).setInt32(0, n, true); // true = little-endian
  return new Uint8Array(buffer);
}
```

</details>

---

## 🔗 Related Topics

- [`12-web-workers.md`](./12-web-workers.md) — SharedArrayBuffer for worker communication
- [`27-proxy-reflect-and-metaprogramming.md`](./27-proxy-reflect-and-metaprogramming.md) — TypedArrays can also be proxied
- [`performance/10-canvas-optimization.md`](../performance/10-canvas-optimization.md) — Canvas pixel manipulation at scale
