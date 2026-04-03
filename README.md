# Dick's Image Resizer, Converter and Upscaler

A single-file, zero-dependency web application for resizing, converting, and AI-upscaling images — entirely in the browser. No server, no uploads, no accounts. Drop an image and go.

---

## What It Does

### Three Core Tabs

**1. Resize**
- Set exact pixel dimensions or scale by percentage
- Lock/unlock aspect ratio with a toggle button
- 15 built-in presets: social media (Instagram Post 1080x1080, Twitter/X 1200x675, YouTube Thumbnail 1280x720, etc.), standard resolutions (HD, Full HD, 4K), and icon sizes (32x32, 64x64, 150x150)
- Downscaling >2:1 ratio uses a **Lanczos-3 resampler in linear light** (same algorithm as ImageMagick/libvips) run inside a Web Worker for non-blocking performance
- Small outputs (<=300px) get automatic sharpening via unsharp mask

**2. Convert**
- 9 output formats: PNG, JPEG, WebP, AVIF, GIF, BMP, TIFF, ICO, SVG
- Quality slider for lossy formats (JPEG, WebP, AVIF)
- Background color picker for transparency-incompatible formats (JPEG, BMP, GIF)
- SVG has two modes:
  - **Wrapped**: embeds the raster image as base64 PNG inside an SVG container
  - **Traced**: true vector conversion using median-cut color quantization + marching squares contour tracing + Ramer-Douglas-Peucker path simplification. Configurable color count (2-64) and detail level (5 settings)
- Custom binary encoders for formats browsers don't natively export: BMP (24-bit uncompressed), GIF (median-cut quantization + LZW compression), TIFF (uncompressed RGBA), ICO (PNG-inside-ICO)

**3. AI Upscale**
- Two neural network models for super-resolution:
  - **Fast (Real-ESRGAN x4plus)**: ~66 MB, runs on main thread with WebGPU acceleration. Best for speed.
  - **Quality (SwinIR-M x4 GAN)**: ~61 MB, runs in a Web Worker with WASM backend. Higher quality textures/detail, ~2x slower.
- Scale factors: 2x, 3x, 4x (single pass), 6x, 8x, 16x (two-pass: 4x then 4x again, then downscale to target)
- Tile-based processing with overlap padding for seamless results at any image size
- RAM estimation per scale factor, with automatic disabling of scales that would exceed available memory
- Models download once from HuggingFace and are cached in IndexedDB — instant on subsequent uses
- Fallback to enhanced upscale (multi-step bicubic + sharpen) if AI models fail to load

### Additional Features

- **Batch processing**: Select multiple files to process all with the same settings. Results download as a .zip archive (JSZip loaded on demand)
- **Before/After comparison**: Draggable divider overlay to compare original vs. processed output
- **EXIF metadata**: Parses and displays camera info (make, model, exposure, ISO, focal length, GPS) for JPEGs. Option to strip metadata on export
- **Edit tools** (history-tracked with undo/redo up to 30 steps):
  - Crop with rule-of-thirds overlay, aspect ratio constraints, and real-time dimension display
  - Rotate (90° CW/CCW, 180°, free angle with canvas expansion)
  - Flip (horizontal/vertical)
  - Adjustments: brightness, contrast, saturation, sharpness, hue, temperature — processed in a Web Worker for large images
  - Filters: grayscale, sepia, invert, vintage, warm, cool, high contrast
- **AI Background Removal**: U2-Net (u2netp) ONNX model, 320x320 input with ImageNet normalization, outputs alpha mask scaled to original resolution
- **Face Enhancement**: Skin-tone detection (RGB heuristic) → bilateral filter on skin regions → sharpen + contrast boost
- **Image Denoising**: Full-image bilateral filter with pre-computed spatial weights, configurable strength
- **Keyboard shortcuts**: Ctrl+S (download), Ctrl+O (open), Ctrl+Enter (process), Escape (cancel)
- **PWA support**: Service worker caches the page and ONNX Runtime assets for offline use
- **Paste from clipboard**: Ctrl+V to paste an image directly

---

## How the Code Works

Everything lives in a single `index.html` file (~3,500 lines): HTML structure, CSS styles, and JavaScript application logic. No build step, no bundler, no framework.

### Architecture Overview

```
index.html
├── <style>          Lines 7-230      CSS (dark theme, responsive grid layout)
├── <script src>     Line 231         ONNX Runtime Web (WebGPU build, from CDN)
├── <body>           Lines 233-449    HTML structure (upload zone, workspace, panels, actions)
├── <script module>  Lines 453-3527   Application JavaScript
│   ├── ONNX Config                   Line 455        WebAssembly paths
│   ├── State (S)                     Lines 460-511   Single global state object
│   ├── Constants                     Lines 513-552   Presets, formats, AI model configs
│   ├── DOM References                Lines 554-563   Cached querySelector results
│   ├── Init & Events                 Lines 565-701   Setup, event binding, tab switching
│   ├── Upload & Workspace            Lines 710-870   File loading, preview, output display
│   ├── Process Router                Lines 873-927   Tab-based dispatch (resize/convert/upscale)
│   ├── Lanczos Resampler             Lines 959-1336  sRGB↔linear, Lanczos-3 kernel, worker + sync fallback
│   ├── Web Worker Infrastructure     Lines 976-999   Generic inline worker creation + promise wrapper
│   ├── ONNX Inference Worker         Lines 1002-1093 SwinIR's dedicated worker thread
│   ├── Convert                       Lines 1338-1373 Format dispatch
│   ├── AI Upscale Pipeline           Lines 1375-1731 Model loading, tile processing, NCHW conversion
│   ├── Non-AI Upscale Fallback       Lines 1733-1768 Multi-step bicubic + sharpen
│   ├── Sharpening                    Lines 1770-1790 Unsharp mask (3x3 kernel)
│   ├── Format Encoders               Lines 1792-2131 BMP, GIF (LZW), TIFF, ICO, SVG
│   ├── Vector Tracing                Lines 2133-2330 Median-cut, marching squares, RDP simplification
│   ├── Edit Tools                    Lines 2332-2816 History, crop, rotate, flip, adjustments, filters
│   ├── Before/After Compare          Lines 2818-2911 Canvas-based split view with draggable divider
│   ├── Batch Processing              Lines 2913-3056 Multi-file processing + JSZip download
│   ├── EXIF Parser                   Lines 3058-3133 Binary TIFF/EXIF tag reader
│   ├── AI Tools                      Lines 3135-3421 Background removal (U2-Net), face enhance, denoise
│   ├── Utilities                     Lines 3435-3493 Download, canvasToBlob, fmtBytes, toast, status
│   └── Service Worker                Lines 3495-3522 PWA offline cache registration
```

### Global State

All application state is a single `const S = {...}` object (line 460). Key fields:

| Field | Purpose |
|-------|---------|
| `file`, `img`, `origW`, `origH` | Loaded image and dimensions |
| `tab` | Active tab: `'resize'` / `'convert'` / `'upscale'` |
| `selectedFmt`, `selectedScale`, `selectedModel` | User selections |
| `ortSession` | Real-ESRGAN ONNX InferenceSession (main thread) |
| `swinirWorker`, `swinirSession` | SwinIR worker handle and loaded flag |
| `editCanvas`, `editCtx`, `history[]`, `historyIdx` | Non-destructive edit state |
| `processing`, `cancelRequested` | Processing lock and cancellation signal |
| `_activeInferenceWorker`, `_cancelTileReject` | Cancel infrastructure for worker mode |

### AI Upscale Pipeline (the most complex subsystem)

**Hybrid inference architecture** — two models, two different execution strategies:

```
Real-ESRGAN (Fast)                    SwinIR (Quality)
─────────────────                     ────────────────
Main thread                           Web Worker
WebGPU backend (GPU-accelerated)      WASM backend (CPU)
session.run() directly                postMessage → worker → result
Cancel via flag + setTimeout yield    Cancel via worker.terminate()
tileSize: 128, no alignment needed    tileSize: 64, align to 8px (window size)
~66 MB ONNX model                     ~61 MB ONNX model
```

**Why the split?** WebGPU's Slice kernel crashes on SwinIR's architecture, so SwinIR must use WASM. WebGPU isn't available in Web Workers, so ESRGAN runs on the main thread for GPU acceleration. The `setTimeout(0)` yield between tiles keeps the UI responsive for cancel.

**Tile processing flow** (`processWithTiles`, line 1611):
1. Divide source image into tiles of `tileSize` pixels
2. For each tile, add `tilePad` pixels of overlap on all sides (clamped to image bounds)
3. For SwinIR, pad dimensions up to the nearest multiple of 8
4. Convert RGBA ImageData → NCHW Float32 tensor (3 channels, values 0.0-1.0)
5. Run inference (produces 4x upscaled output)
6. Convert NCHW Float32 → RGBA ImageData
7. Crop off the overlap padding from the upscaled output
8. Composite the cropped tile into the output canvas at the correct position
9. Repeat for all tiles, updating progress bar

**Multi-pass for >4x**: The models natively produce 4x output. For 6x/8x/16x, the pipeline runs two 4x passes (producing 16x), then downscales to the exact target using high-quality canvas interpolation.

**Model loading** (`ensureAI`, line 1403):
1. Check if model is already loaded (return early)
2. Check IndexedDB cache for the model binary
3. If not cached, stream-download from HuggingFace with progress reporting
4. Cache the ArrayBuffer in IndexedDB for next time
5. For ESRGAN: create `ort.InferenceSession` on main thread with `['webgpu', 'wasm']` providers
6. For SwinIR: create a Web Worker blob, send it the model buffer, worker creates its own session
7. Store session/worker references in state

### Lanczos-3 Resampling (line 1096)

Used for high-quality downscaling (>2:1 ratio). Runs as a **separable 2-pass filter in linear light**:

1. **sRGB → Linear**: Pre-computed 256-entry lookup table for the gamma transfer function
2. **Horizontal pass**: For each output pixel, compute weighted average of source pixels using the Lanczos-3 kernel (sinc windowed by sinc, support radius 3)
3. **Vertical pass**: Same operation on the horizontal pass output
4. **Linear → sRGB**: Inverse gamma for each output channel

The heavy lifting runs in a Web Worker with progress reporting. A synchronous fallback exists if the worker fails.

### Format Encoders

**BMP** (line 1792): Constructs the 54-byte header (BITMAPINFOHEADER, top-down, 24-bit) and writes BGR pixel data with row padding to 4-byte boundaries.

**GIF** (line 1834): Full pipeline: median-cut color quantization (recursive bucket splitting by widest channel range) → nearest-color index mapping → LZW variable-width code compression → GIF89a binary with sub-block framing.

**TIFF** (line 1999): Little-endian TIFF with 12 IFD tags (width, height, bits per sample, compression=none, RGB photometric, 4 samples/pixel with unassociated alpha). Pixel data written as raw RGBA.

**ICO** (line 2062): Scales image to max 256x256, renders as PNG, wraps in ICO container (ICONDIR + ICONDIRENTRY + PNG payload).

**SVG Trace** (line 2134): Median-cut quantization → per-color binary mask → marching squares contour tracing (16-cell lookup with saddle disambiguation) → Ramer-Douglas-Peucker path simplification → SVG path elements with fill colors.

### CSS / UI Design

Dark theme with CSS custom properties (line 9). Layout uses CSS Grid for format/scale selectors, Flexbox for rows and actions. Responsive breakpoint at 640px collapses the two-column preview to single column. Animations: spin (spinner), pulse (loading dots), slideIn (toasts), fadeIn (workspace), indeterminate (progress bar). The checkerboard transparency pattern uses `repeating-conic-gradient`.

### External Dependencies

| Dependency | Loaded From | Purpose |
|-----------|-------------|---------|
| ONNX Runtime Web 1.21.0 | jsdelivr CDN | AI model inference (WebGPU + WASM) |
| JSZip 3.x | jsdelivr CDN | Batch download as .zip (loaded on demand) |

Both are loaded from CDN — there are no local dependencies or `node_modules`.
