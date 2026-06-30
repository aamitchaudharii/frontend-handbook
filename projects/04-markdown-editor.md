# 04 — Project: Markdown Editor with Live Preview

> **"A markdown editor seems like a textarea with a parser attached, until you need synchronized scrolling between editor and preview, undo/redo that doesn't fight the browser's native textarea undo stack, autosave that doesn't thrash the network, and a preview that re-renders fast enough to feel live without re-parsing the entire document on every keystroke. Each of these is a small architecture decision that compounds."**

This project guide builds a markdown editor: a split-pane editor/preview, synchronized scrolling, debounced autosave with conflict detection, keyboard shortcuts for formatting, image paste/drag-drop upload, and incremental parsing for performance on long documents.

---

## 📚 What You'll Build

A markdown editor with: a text editing pane and a live-rendering preview pane, synchronized scroll position between them, autosave with visual save-status feedback, formatting keyboard shortcuts (bold, italic, links), drag-and-drop image upload, and a document outline sidebar generated from headings.

---

## Requirements

```
FUNCTIONAL:
  - Split-pane editor + live preview (toggleable to single-pane modes)
  - Synchronized scrolling between editor and preview
  - Autosave with debouncing, visual save status indicator
  - Keyboard shortcuts: Cmd/Ctrl+B (bold), Cmd/Ctrl+I (italic), Cmd/Ctrl+K (link)
  - Image upload via paste or drag-and-drop, inserted as markdown image syntax
  - Document outline (table of contents) generated from heading structure
  - Export to HTML and raw markdown

NON-FUNCTIONAL:
  - Preview re-render must not cause visible lag while typing, even on
    documents of 10,000+ words
  - Must not lose unsaved content on accidental tab close (beforeunload warning)
  - Undo/redo must feel native (use the browser's built-in textarea undo
    stack rather than reimplementing it, where possible)
```

---

## Architecture Overview

```
COMPONENT TREE:
  <EditorPage>
    <EditorToolbar />            (formatting buttons, view mode toggle)
    <EditorSplitView>
      <MarkdownEditor />          (textarea or CodeMirror instance)
      <MarkdownPreview />         (rendered HTML output)
    <DocumentOutline />          (sidebar: heading-based TOC)
    <SaveStatusIndicator />      (saving / saved / error)

KEY ARCHITECTURE DECISIONS:
  1. Editor: plain <textarea> vs CodeMirror/Monaco
     For markdown specifically, a plain textarea with custom keyboard
     handlers is often SUFFICIENT and avoids a heavy editor library
     dependency — reserve CodeMirror/Monaco for cases needing syntax
     highlighting WITHIN the editor itself (not just the preview).

  2. Parsing strategy: parse the FULL document on every change vs
     incremental/debounced parsing — addressed in Step 2.

  3. Scroll sync: proportional scroll position vs line-mapped scroll
     position — addressed in Step 3.
```

---

## Step 1 — The Editor Pane (Textarea-Based)

```jsx
function MarkdownEditor({ value, onChange, onScroll }) {
  const textareaRef = useRef(null);

  function handleKeyDown(e) {
    const isCmd = e.metaKey || e.ctrlKey;

    if (isCmd && e.key === "b") {
      e.preventDefault();
      wrapSelection(textareaRef.current, "**", "**");
    }
    if (isCmd && e.key === "i") {
      e.preventDefault();
      wrapSelection(textareaRef.current, "_", "_");
    }
    if (isCmd && e.key === "k") {
      e.preventDefault();
      wrapSelection(textareaRef.current, "[", "](url)");
    }
    // Tab key: insert spaces instead of moving focus (common editor expectation)
    if (e.key === "Tab") {
      e.preventDefault();
      insertAtCursor(textareaRef.current, "  ");
    }
  }

  function wrapSelection(textarea, before, after) {
    const { selectionStart, selectionEnd, value } = textarea;
    const selected = value.slice(selectionStart, selectionEnd);
    const newValue =
      value.slice(0, selectionStart) +
      before +
      selected +
      after +
      value.slice(selectionEnd);

    // Use the native execCommand-free approach: directly manipulate value
    // via the textarea's setRangeText, which PRESERVES the undo stack
    // (unlike directly setting .value, which clears undo history)
    textarea.setRangeText(
      before + selected + after,
      selectionStart,
      selectionEnd,
      "select",
    );
    onChange(textarea.value);

    // Reposition cursor inside the wrapping syntax if no text was selected
    if (!selected) {
      const cursorPos = selectionStart + before.length;
      textarea.setSelectionRange(cursorPos, cursorPos);
    }
  }

  return (
    <textarea
      ref={textareaRef}
      value={value}
      onChange={(e) => onChange(e.target.value)}
      onKeyDown={handleKeyDown}
      onScroll={onScroll}
      className="markdown-editor"
      spellCheck={true}
      placeholder="Start writing..."
    />
  );
}
```

**Key decision:** using `textarea.setRangeText()` instead of directly setting `textarea.value = newValue` is what preserves the browser's NATIVE undo/redo stack (Cmd/Ctrl+Z). Directly overwriting `.value` programmatically clears the undo history, which would make Cmd+Z after a bold/italic shortcut undo the ENTIRE document instead of just that formatting action — a jarring, broken-feeling experience. `setRangeText` is purpose-built for exactly this kind of programmatic text manipulation that should still participate in the native undo stack.

---

## Step 2 — Incremental/Debounced Parsing for Performance

```typescript
// NAIVE: re-parse and re-render the ENTIRE document on every keystroke
// For a 10,000-word document, this can take 20-50ms per keystroke —
// noticeably laggy, especially combined with React's re-render cost

// BETTER: debounce parsing, but keep typing itself instant
function useMarkdownPreview(markdown: string, debounceMs = 150) {
  const [renderedHtml, setRenderedHtml] = useState("");
  const [isStale, setIsStale] = useState(false);

  useEffect(() => {
    setIsStale(true); // preview is now out of sync with the editor

    const timer = setTimeout(() => {
      const html = parseMarkdown(markdown); // expensive operation
      setRenderedHtml(html);
      setIsStale(false);
    }, debounceMs);

    return () => clearTimeout(timer);
  }, [markdown, debounceMs]);

  return { renderedHtml, isStale };
}

// EVEN BETTER for very large documents: parse in a Web Worker to avoid
// blocking the main thread (and therefore typing responsiveness) entirely
function useMarkdownPreviewWorker(markdown: string, debounceMs = 150) {
  const [renderedHtml, setRenderedHtml] = useState("");
  const workerRef = useRef<Worker | null>(null);

  useEffect(() => {
    workerRef.current = new Worker(
      new URL("./markdown-worker.js", import.meta.url),
    );
    workerRef.current.onmessage = (e) => setRenderedHtml(e.data.html);
    return () => workerRef.current?.terminate();
  }, []);

  useEffect(() => {
    const timer = setTimeout(() => {
      workerRef.current?.postMessage({ markdown });
    }, debounceMs);
    return () => clearTimeout(timer);
  }, [markdown, debounceMs]);

  return renderedHtml;
}

// markdown-worker.js
import { marked } from "marked"; // or your markdown parser of choice
self.onmessage = (e) => {
  const html = marked.parse(e.data.markdown);
  self.postMessage({ html });
};
```

**Key decision:** the debounce (150ms) means the preview doesn't update on EVERY keystroke, only after a brief pause — this is the difference between "live preview" and "preview that fights you for CPU time while typing." For documents where even debounced main-thread parsing causes jank (very long documents, complex markdown extensions), moving the parse itself to a Web Worker removes it from the main thread entirely, keeping typing perfectly smooth regardless of document size.

---

## Step 3 — Synchronized Scrolling

```typescript
// Proportional scroll sync: map scroll percentage, not exact pixel position
// (editor and preview have DIFFERENT total heights, so pixel-for-pixel
// mapping would desync quickly)

function useSyncedScroll(editorRef, previewRef) {
  const isSyncingRef = useRef(false); // prevent infinite scroll-sync loops

  function handleEditorScroll() {
    if (isSyncingRef.current) {
      isSyncingRef.current = false;
      return;
    }

    const editor = editorRef.current;
    const preview = previewRef.current;

    const scrollPercentage =
      editor.scrollTop / (editor.scrollHeight - editor.clientHeight);

    isSyncingRef.current = true;
    preview.scrollTop =
      scrollPercentage * (preview.scrollHeight - preview.clientHeight);
  }

  function handlePreviewScroll() {
    if (isSyncingRef.current) {
      isSyncingRef.current = false;
      return;
    }

    const editor = editorRef.current;
    const preview = previewRef.current;

    const scrollPercentage =
      preview.scrollTop / (preview.scrollHeight - preview.clientHeight);

    isSyncingRef.current = true;
    editor.scrollTop =
      scrollPercentage * (editor.scrollHeight - editor.clientHeight);
  }

  return { handleEditorScroll, handlePreviewScroll };
}
```

**Key decision:** the `isSyncingRef` flag prevents an infinite loop — without it, scrolling the editor triggers a preview scroll, which fires the preview's onScroll handler, which scrolls the editor again, which fires the editor's onScroll handler again, forever. This guard ensures each programmatic scroll update is "consumed" by the flag check and doesn't trigger another round of synchronization.

**Limitation worth knowing:** proportional (percentage-based) scroll sync is simple but imprecise — it doesn't guarantee that the heading you're looking at in the editor is the SAME heading visible in the preview, especially in documents with very uneven content density (e.g., a huge code block followed by a single line of text). Production editors (like Typora, HackMD) use line-to-DOM-node mapping for precise sync, which requires tracking which markdown line corresponds to which rendered DOM element — a meaningfully more complex but more accurate approach.

---

## Step 4 — Autosave with Status Feedback and Conflict Awareness

```typescript
type SaveStatus = 'idle' | 'saving' | 'saved' | 'error' | 'conflict';

function useAutosave(documentId: string, content: string, debounceMs = 1000) {
  const [status, setStatus] = useState<SaveStatus>('idle');
  const lastSavedContent = useRef(content);
  const lastKnownVersion = useRef<number | null>(null);

  useEffect(() => {
    if (content === lastSavedContent.current) return; // no actual change

    setStatus('saving');
    const timer = setTimeout(async () => {
      try {
        const response = await documentsApi.save(documentId, {
          content,
          expectedVersion: lastKnownVersion.current, // optimistic concurrency check
        });

        lastSavedContent.current = content;
        lastKnownVersion.current = response.version;
        setStatus('saved');
      } catch (err) {
        if (err.code === 'VERSION_CONFLICT') {
          // Someone else saved a different version since we last loaded
          setStatus('conflict');
        } else {
          setStatus('error');
        }
      }
    }, debounceMs);

    return () => clearTimeout(timer);
  }, [content, documentId, debounceMs]);

  // Warn before leaving with unsaved changes
  useEffect(() => {
    function handleBeforeUnload(e: BeforeUnloadEvent) {
      if (status === 'saving' || content !== lastSavedContent.current) {
        e.preventDefault();
        e.returnValue = ''; // required for the browser to show the confirmation dialog
      }
    }
    window.addEventListener('beforeunload', handleBeforeUnload);
    return () => window.removeEventListener('beforeunload', handleBeforeUnload);
  }, [status, content]);

  return status;
}

function SaveStatusIndicator({ status }) {
  const labels = {
    idle:     '',
    saving:   'Saving...',
    saved:    'All changes saved',
    error:    'Failed to save — retrying...',
    conflict: 'This document was changed elsewhere — please refresh',
  };

  return <span className={`save-status save-status--${status}`}>{labels[status]}</span>;
}
```

**Key decision:** the `expectedVersion` field implements optimistic concurrency control — the server rejects the save if the document's current version doesn't match what the client last fetched, indicating someone else saved a change in between. This prevents the classic "last write wins, silently overwriting someone else's edits" problem that naive autosave implementations have, surfacing a `conflict` status instead so the user can be warned rather than unknowingly destroying another user's work.

---

## Step 5 — Document Outline from Heading Structure

```typescript
function useDocumentOutline(markdown: string) {
  return useMemo(() => {
    const headingRegex = /^(#{1,6})\s+(.+)$/gm;
    const headings: { level: number; text: string; slug: string }[] = [];

    let match;
    while ((match = headingRegex.exec(markdown)) !== null) {
      const level = match[1].length;
      const text  = match[2].trim();
      const slug  = text.toLowerCase().replace(/[^\w]+/g, '-');
      headings.push({ level, text, slug });
    }

    return headings;
  }, [markdown]);
}

function DocumentOutline({ markdown, onHeadingClick }) {
  const outline = useDocumentOutline(markdown);

  return (
    <nav className="document-outline" aria-label="Document outline">
      {outline.map((heading, i) => (
        <button
          key={i}
          className={`outline-item outline-item--level-${heading.level}`}
          onClick={() => onHeadingClick(heading.slug)}
        >
          {heading.text}
        </button>
      ))}
    </nav>
  );
}
```

**Key decision:** the outline is recomputed via `useMemo` keyed on `markdown` — but since `markdown` changes on every keystroke, this would re-run the regex on every keystroke too. For large documents this regex scan is cheap relative to full markdown parsing, but if it becomes measurable, debounce it the same way the preview rendering is debounced in Step 2 (the two share the same fundamental "keystroke vs background processing" tradeoff).

---

## Step 6 — Image Paste and Drag-Drop Upload

```typescript
function useImageUpload(textareaRef, onChange) {
  async function uploadAndInsert(file: File) {
    const textarea = textareaRef.current;
    const cursorPos = textarea.selectionStart;

    // Insert a placeholder immediately for instant feedback
    const placeholder = `![Uploading ${file.name}...]()`;
    const valueWithPlaceholder =
      textarea.value.slice(0, cursorPos) +
      placeholder +
      textarea.value.slice(cursorPos);
    onChange(valueWithPlaceholder);

    try {
      const url = await uploadImage(file); // your upload API call
      const markdownImage = `![${file.name}](${url})`;

      // Replace the placeholder with the real markdown image syntax
      onChange((prev) => prev.replace(placeholder, markdownImage));
    } catch (err) {
      onChange((prev) =>
        prev.replace(placeholder, `![Upload failed: ${file.name}]()`),
      );
    }
  }

  function handlePaste(e: React.ClipboardEvent) {
    const items = Array.from(e.clipboardData.items);
    const imageItem = items.find((item) => item.type.startsWith("image/"));
    if (imageItem) {
      e.preventDefault();
      const file = imageItem.getAsFile();
      if (file) uploadAndInsert(file);
    }
  }

  function handleDrop(e: React.DragEvent) {
    e.preventDefault();
    const file = Array.from(e.dataTransfer.files).find((f) =>
      f.type.startsWith("image/"),
    );
    if (file) uploadAndInsert(file);
  }

  return { handlePaste, handleDrop };
}
```

**Key decision:** inserting a placeholder (`![Uploading filename...]()`) immediately, then replacing it once the upload completes, gives the user instant feedback that their paste/drop was recognized — without this, there'd be a confusing delay where nothing visibly happens while the upload is in flight, and users might paste/drop again, creating duplicates.

---

## Performance Checklist

```
☐ Preview parsing is debounced, not synchronous on every keystroke
☐ Very large documents (10,000+ words): consider moving parsing to a Web Worker
☐ Formatting shortcuts use setRangeText (preserves native undo stack)
☐ Autosave is debounced to avoid excessive network requests
☐ Document outline regex scan doesn't block typing on large documents
```

## Reliability Checklist

```
☐ beforeunload warning when there are unsaved changes
☐ Optimistic concurrency control prevents silent overwrite conflicts
☐ Save status clearly communicated (saving/saved/error/conflict)
☐ Failed saves retry automatically with backoff
```

---

## Extension Ideas

```
- Collaborative real-time editing (would require CRDT/OT — see
  projects/01-realtime-chat-application.md's WebSocket patterns as a
  starting point, combined with an operational transform library)
- Syntax highlighting within code blocks in the preview
- Custom markdown extensions (footnotes, callout boxes, embeds)
- Version history with diff view between versions
- Export to PDF
- Slash-command menu for inserting markdown elements
```

---

## 🔗 Related Topics

- [`javascript-core/14-web-workers.md`](../javascript-core/14-web-workers.md) — Offloading parsing to workers
- [`performance/04-raf-optimization.md`](../performance/04-raf-optimization.md) — Debouncing patterns
- [`patterns/02-custom-hooks.md`](../patterns/02-custom-hooks.md) — Hook design used throughout

---

<div align="center">

**Next:** [`projects/05-analytics-dashboard.md`](./05-analytics-dashboard.md) →

</div>
