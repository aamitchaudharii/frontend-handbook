# 10 — Project: File Upload System

> **"File uploads are where the network's unreliability stops being an abstraction and becomes the user's problem directly — a dropped connection 80% through a 2GB video upload is not a hypothetical edge case, it's Tuesday. A file upload system's quality is measured almost entirely by how gracefully it behaves when something goes wrong, not by how it behaves on a fast, stable connection."**

This project guide builds a robust file upload system: drag-and-drop with validation, chunked uploads for large files with resume support, progress tracking, client-side image compression, and concurrent upload management with retry — covering the failure modes that separate a toy upload widget from a production-grade one.

---

## 📚 What You'll Build

An upload system with: drag-and-drop and click-to-browse file selection, client-side validation (type, size, dimensions), chunked upload for large files with pause/resume capability, concurrent upload queueing with a configurable limit, client-side image compression before upload, and per-file progress with retry on failure.

---

## Requirements

```
FUNCTIONAL:
  - Drag-and-drop and traditional file picker input
  - Multi-file upload with per-file progress
  - Validation before upload starts: file type, size limits, image dimensions
  - Large file support (500MB+) via chunked upload
  - Resume an interrupted upload without restarting from zero
  - Client-side image compression/resizing before upload (reduce bandwidth)
  - Cancel individual uploads in progress

NON-FUNCTIONAL:
  - Network interruption mid-upload must not corrupt the upload or
    require a full restart for large files
  - Uploading 20 files simultaneously should not open 20 simultaneous
    connections (concurrency limiting)
  - Memory usage must stay reasonable even for large files (don't load
    an entire 2GB file into memory at once)
```

---

## Architecture Overview

```
COMPONENT TREE:
  <FileUploadZone>               (drag-drop target + file picker trigger)
    <FileList>
      <FileUploadItem />          (per-file: thumbnail, progress, status, actions)

UPLOAD PIPELINE PER FILE:
  1. Validate (type, size, dimensions)
  2. Optionally compress (images)
  3. Queue (respect concurrency limit)
  4. Upload (chunked for large files, single request for small ones)
  5. Confirm (server returns final URL/metadata)

KEY ARCHITECTURE DECISION: uploads are chunked using the File API's
Blob.slice() method, with each chunk uploaded as a separate request,
allowing pause/resume by tracking which chunks have been successfully
received by the server.
```

---

## Step 1 — Validation Before Upload Starts

```typescript
interface ValidationRule {
  validate: (file: File) => Promise<string | null>; // returns error message or null
}

const maxSizeRule = (maxBytes: number): ValidationRule => ({
  validate: async (file) =>
    file.size > maxBytes
      ? `File exceeds maximum size of ${formatBytes(maxBytes)}`
      : null,
});

const allowedTypesRule = (types: string[]): ValidationRule => ({
  validate: async (file) =>
    types.includes(file.type)
      ? null
      : `File type ${file.type} is not supported`,
});

const imageDimensionsRule = (
  maxWidth: number,
  maxHeight: number,
): ValidationRule => ({
  validate: async (file) => {
    if (!file.type.startsWith("image/")) return null; // skip for non-images

    const dimensions = await getImageDimensions(file);
    if (dimensions.width > maxWidth || dimensions.height > maxHeight) {
      return `Image dimensions (${dimensions.width}×${dimensions.height}) exceed maximum (${maxWidth}×${maxHeight})`;
    }
    return null;
  },
});

function getImageDimensions(
  file: File,
): Promise<{ width: number; height: number }> {
  return new Promise((resolve, reject) => {
    const img = new Image();
    const url = URL.createObjectURL(file);
    img.onload = () => {
      resolve({ width: img.naturalWidth, height: img.naturalHeight });
      URL.revokeObjectURL(url); // release memory immediately after reading dimensions
    };
    img.onerror = () => {
      URL.revokeObjectURL(url);
      reject(new Error("Failed to load image"));
    };
    img.src = url;
  });
}

async function validateFile(
  file: File,
  rules: ValidationRule[],
): Promise<string[]> {
  const results = await Promise.all(rules.map((rule) => rule.validate(file)));
  return results.filter((error): error is string => error !== null);
}
```

**Key decision:** `URL.createObjectURL()` is revoked IMMEDIATELY after reading the image's dimensions, not held onto — object URLs keep their underlying blob alive in memory until explicitly revoked or the document unloads, so forgetting this for a batch of large image uploads would be a meaningful, easily-overlooked memory leak (see [`anti-patterns/05-memory-leaks.md`](../anti-patterns/05-memory-leaks.md)).

---

## Step 2 — Client-Side Image Compression

```typescript
async function compressImage(
  file: File,
  { maxWidth = 1920, maxHeight = 1920, quality = 0.8 } = {},
): Promise<File> {
  if (!file.type.startsWith("image/") || file.type === "image/gif") {
    return file; // skip non-images and GIFs (compression would break animation)
  }

  const bitmap = await createImageBitmap(file); // efficient decode, off the main render path

  let { width, height } = bitmap;
  if (width > maxWidth || height > maxHeight) {
    const scale = Math.min(maxWidth / width, maxHeight / height);
    width = Math.round(width * scale);
    height = Math.round(height * scale);
  }

  const canvas = new OffscreenCanvas(width, height); // off-main-thread-capable canvas
  const ctx = canvas.getContext("2d")!;
  ctx.drawImage(bitmap, 0, 0, width, height);
  bitmap.close(); // release the decoded bitmap's memory promptly

  const blob = await canvas.convertToBlob({ type: "image/jpeg", quality });

  return new File([blob], file.name, { type: "image/jpeg" });
}

// Usage: compress before adding to the upload queue
async function prepareFileForUpload(file: File): Promise<File> {
  if (file.type.startsWith("image/") && file.size > 1_000_000) {
    // only compress if > 1MB
    return compressImage(file);
  }
  return file;
}
```

**Key decision:** `createImageBitmap` and `OffscreenCanvas` are used instead of the traditional `<img>` + `<canvas>` DOM approach — this entire compression pipeline can run WITHOUT touching the DOM at all, and `OffscreenCanvas` specifically supports running inside a Web Worker, meaning the (potentially CPU-intensive) compression of many large images can be moved off the main thread entirely if it becomes a bottleneck, with no architecture changes needed beyond moving this function into a worker.

---

## Step 3 — Chunked Upload with Resume Support

```typescript
const CHUNK_SIZE = 5 * 1024 * 1024; // 5MB per chunk

interface ChunkedUploadState {
  fileId: string;
  totalChunks: number;
  uploadedChunks: Set<number>; // which chunk indices have been confirmed by the server
}

class ChunkedUploader {
  #file: File;
  #fileId: string;
  #uploadedChunks = new Set<number>();
  #totalChunks: number;
  #abortController = new AbortController();
  #onProgress: (percent: number) => void;

  constructor(
    file: File,
    fileId: string,
    onProgress: (percent: number) => void,
  ) {
    this.#file = file;
    this.#fileId = fileId;
    this.#totalChunks = Math.ceil(file.size / CHUNK_SIZE);
    this.#onProgress = onProgress;
  }

  // Check with the server which chunks (if any) were already received
  // from a PREVIOUS attempt — this is what makes resume possible
  async resumeIfPossible() {
    try {
      const status = await uploadApi.getUploadStatus(this.#fileId);
      this.#uploadedChunks = new Set(status.receivedChunkIndices);
    } catch {
      this.#uploadedChunks = new Set(); // no previous attempt found — start fresh
    }
  }

  async upload() {
    await this.resumeIfPossible();

    for (let i = 0; i < this.#totalChunks; i++) {
      if (this.#uploadedChunks.has(i)) {
        this.#reportProgress(); // already uploaded — count it, skip re-sending
        continue;
      }

      if (this.#abortController.signal.aborted) return;

      await this.#uploadChunk(i);
      this.#uploadedChunks.add(i);
      this.#reportProgress();
    }

    return uploadApi.finalizeUpload(this.#fileId); // tells server all chunks received, assemble file
  }

  async #uploadChunk(index: number, retriesLeft = 3) {
    const start = index * CHUNK_SIZE;
    const chunk = this.#file.slice(start, start + CHUNK_SIZE); // doesn't load whole file into memory

    try {
      await uploadApi.uploadChunk({
        fileId: this.#fileId,
        chunkIndex: index,
        totalChunks: this.#totalChunks,
        data: chunk,
        signal: this.#abortController.signal,
      });
    } catch (err) {
      if (this.#abortController.signal.aborted) return; // intentional cancellation, don't retry
      if (retriesLeft > 0) {
        await new Promise((r) => setTimeout(r, 1000)); // brief backoff before retry
        return this.#uploadChunk(index, retriesLeft - 1);
      }
      throw err; // exhausted retries — propagate failure
    }
  }

  #reportProgress() {
    const percent = (this.#uploadedChunks.size / this.#totalChunks) * 100;
    this.#onProgress(percent);
  }

  pause() {
    this.#abortController.abort();
  }

  // Resuming after pause: create a NEW ChunkedUploader instance with the
  // same fileId — resumeIfPossible() will pick up where it left off
  // because the server tracks received chunks by fileId
}
```

**Key decision:** `File.slice()` is what makes chunking memory-efficient — it creates a `Blob` referencing a byte range of the original file WITHOUT reading that range into memory until actually consumed (e.g., by `FormData` or `fetch`'s body). This is why a 2GB file can be chunked and uploaded without the browser ever holding the entire 2GB in memory at once — only one 5MB chunk's worth of data is actually read and transmitted at a time. Resume works because the server is the source of truth for "which chunks have I actually received" (`getUploadStatus`), not any client-side state that could be lost if the browser tab crashes mid-upload.

---

## Step 4 — Concurrency-Limited Upload Queue

```typescript
class UploadQueue {
  #concurrencyLimit: number;
  #activeCount = 0;
  #pending: Array<() => Promise<void>> = [];

  constructor(concurrencyLimit = 3) {
    this.#concurrencyLimit = concurrencyLimit;
  }

  add(uploadTask: () => Promise<void>): Promise<void> {
    return new Promise((resolve, reject) => {
      const run = async () => {
        this.#activeCount++;
        try {
          await uploadTask();
          resolve();
        } catch (err) {
          reject(err);
        } finally {
          this.#activeCount--;
          this.#processNext();
        }
      };

      if (this.#activeCount < this.#concurrencyLimit) {
        run();
      } else {
        this.#pending.push(run);
      }
    });
  }

  #processNext() {
    if (
      this.#pending.length > 0 &&
      this.#activeCount < this.#concurrencyLimit
    ) {
      const next = this.#pending.shift()!;
      next();
    }
  }
}

// Usage: uploading 20 files only runs 3 at a time
const uploadQueue = new UploadQueue(3);

function uploadFiles(files: File[]) {
  return files.map((file) => {
    const uploader = new ChunkedUploader(file, generateFileId(), (percent) => {
      updateFileProgress(file, percent);
    });

    return uploadQueue.add(() => uploader.upload());
  });
}
```

**Key decision:** without a concurrency limit, selecting 20 files for upload would open 20 simultaneous HTTP connections (or worse, 20 × however-many-chunks-each in a naive implementation) — most browsers cap concurrent connections per origin at around 6, meaning the EXCESS requests would simply queue at the browser/network layer anyway, but with no application-level visibility or control over WHICH uploads get priority. An explicit queue with a deliberate limit gives predictable behavior and lets you reason about and display "3 uploading, 17 waiting" rather than a chaotic burst.

---

## Step 5 — The React Integration

```jsx
function useFileUpload() {
  const [files, setFiles] = useState<UploadFile[]>([]);
  const uploadersRef = useRef<Map<string, ChunkedUploader>>(new Map());

  async function addFiles(fileList: FileList) {
    const validationRules = [
      maxSizeRule(500 * 1024 * 1024), // 500MB
      allowedTypesRule(['image/jpeg', 'image/png', 'video/mp4']),
    ];

    for (const file of Array.from(fileList)) {
      const errors = await validateFile(file, validationRules);
      const id = crypto.randomUUID();

      if (errors.length > 0) {
        setFiles(prev => [...prev, { id, file, status: 'invalid', errors, progress: 0 }]);
        continue;
      }

      const processedFile = await prepareFileForUpload(file); // compress if applicable
      setFiles(prev => [...prev, { id, file: processedFile, status: 'queued', progress: 0 }]);

      const uploader = new ChunkedUploader(processedFile, id, (percent) => {
        setFiles(prev => prev.map(f => f.id === id ? { ...f, progress: percent, status: 'uploading' } : f));
      });
      uploadersRef.current.set(id, uploader);

      uploadQueue.add(() => uploader.upload())
        .then(() => setFiles(prev => prev.map(f => f.id === id ? { ...f, status: 'done' } : f)))
        .catch(() => setFiles(prev => prev.map(f => f.id === id ? { ...f, status: 'failed' } : f)));
    }
  }

  function cancelUpload(id: string) {
    uploadersRef.current.get(id)?.pause();
    setFiles(prev => prev.map(f => f.id === id ? { ...f, status: 'cancelled' } : f));
  }

  function retryUpload(id: string) {
    const file = files.find(f => f.id === id);
    if (!file) return;
    setFiles(prev => prev.map(f => f.id === id ? { ...f, status: 'queued' } : f));
    const uploader = new ChunkedUploader(file.file, id, (percent) => {
      setFiles(prev => prev.map(f => f.id === id ? { ...f, progress: percent } : f));
    });
    uploadQueue.add(() => uploader.upload())
      .then(() => setFiles(prev => prev.map(f => f.id === id ? { ...f, status: 'done' } : f)))
      .catch(() => setFiles(prev => prev.map(f => f.id === id ? { ...f, status: 'failed' } : f)));
  }

  return { files, addFiles, cancelUpload, retryUpload };
}
```

---

## Step 6 — Drag and Drop Zone

```jsx
function FileUploadZone({ onFilesAdded }) {
  const [isDragging, setIsDragging] = useState(false);
  const fileInputRef = useRef(null);

  function handleDrop(e) {
    e.preventDefault();
    setIsDragging(false);
    onFilesAdded(e.dataTransfer.files);
  }

  return (
    <div
      className={`upload-zone ${isDragging ? "upload-zone--dragging" : ""}`}
      onDragOver={(e) => {
        e.preventDefault();
        setIsDragging(true);
      }}
      onDragLeave={() => setIsDragging(false)}
      onDrop={handleDrop}
      onClick={() => fileInputRef.current.click()}
      role="button"
      tabIndex={0}
      onKeyDown={(e) => {
        if (e.key === "Enter" || e.key === " ") fileInputRef.current.click();
      }}
    >
      <input
        ref={fileInputRef}
        type="file"
        multiple
        hidden
        onChange={(e) => onFilesAdded(e.target.files)}
      />
      <UploadIcon />
      <p>Drag and drop files here, or click to browse</p>
    </div>
  );
}
```

**Key decision:** the drop zone is also keyboard-accessible (`role="button"`, `tabIndex={0}`, Enter/Space triggers the hidden file input) — drag-and-drop is fundamentally a pointer-only interaction, so without this fallback, keyboard-only users would have no way to trigger file selection at all.

---

## Performance and Reliability Checklist

```
☐ Large files chunked via File.slice() — never load an entire large
  file into memory at once
☐ Image compression uses createImageBitmap + OffscreenCanvas (DOM-free,
  worker-compatible)
☐ Object URLs revoked immediately after use
☐ Upload concurrency explicitly limited and queued
☐ Resume works by querying SERVER state for received chunks, not
  trusting client-side memory of progress
☐ Failed chunks retry with backoff before failing the whole upload
☐ Drag-and-drop zone has a keyboard-accessible fallback
```

---

## Extension Ideas

```
- Direct-to-cloud-storage uploads (presigned URLs to S3/GCS, bypassing
  your own server for the file bytes entirely)
- Background upload that survives navigation away from the upload page
  (Service Worker + Background Sync API)
- Video transcoding status polling (for video uploads requiring server-side processing)
- Duplicate file detection via client-side hashing before upload
- Folder upload support (webkitdirectory attribute)
```

---

## 🔗 Related Topics

- [`anti-patterns/05-memory-leaks.md`](../anti-patterns/05-memory-leaks.md) — Object URL cleanup patterns
- [`networking/02-rest-and-graphql.md`](../networking/02-rest-and-graphql.md) — Upload API design considerations
- [`patterns/02-custom-hooks.md`](../patterns/02-custom-hooks.md) — Hook design used throughout

---

<div align="center">

**Next:** [`projects/11-component-library.md`](./11-component-library.md) →

</div>
