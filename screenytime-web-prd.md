# Screenytime Web — Product Requirements Document

## Overview

Screenytime is a browser-based tool for wrapping phone screen recordings in a device frame with a custom background color. It runs entirely client-side (no server processing). It will be hosted at **ballrs.net/screenytime** as a page within the existing Vercel-hosted site.

The tool replicates what the existing `screenytime.sh` bash script does, but with a visual UI so non-technical users can operate it.

---

## IMPORTANT: Deployment Context

The ballrs.net website is a **static HTML/CSS site** hosted on Vercel. It is NOT a Next.js project. The site lives at:

```
~/Projects/ballrs-website/
```

**Build this as a single self-contained `screenytime.html` file with all JavaScript inline.** No frameworks, no build step, no dependencies. Just HTML, CSS, and vanilla JS in one file.

The two image assets (`device-frame.png` and `cover-up.png`) go in the same root directory alongside the HTML file.

**Final file structure:**
```
ballrs-website/
├── screenytime.html      ← NEW (all JS/CSS inline, single file)
├── device-frame.png      ← NEW (copy from ~/Projects/screenytime/)
├── cover-up.png          ← NEW (copy from ~/Projects/screenytime/)
├── index.html
├── delete-account.html
├── privacy-policy.html
├── terms-of-service.html
├── styles.css
├── vercel.json
└── ... (existing files)
```

Vercel serves `.html` files without the extension, so the page will be accessible at **ballrs.net/screenytime**.

---

## What It Does

1. User uploads a screen recording (MP4/MOV from iPhone)
2. User picks a background color
3. The tool composites: colored background → screen recording (with rounded corners) → device frame PNG → cover-up overlay
4. Optionally: speed up the video (1x, 1.5x, 2x)
5. Optionally: concatenate an end card video after the device video
6. User downloads the final MP4

All processing happens in the browser using the **Canvas API** and **MediaRecorder API**. No uploads to any server. No third-party dependencies for video processing.

---

## Tech Stack

- **Single HTML file** with inline CSS and JavaScript
- **HTML5 Canvas API** for frame-by-frame video compositing
- **MediaRecorder API** for encoding the canvas output to MP4/WebM
- **Vanilla JavaScript** — no frameworks, no libraries, no build step

---

## UI Layout

Single page, vertically stacked sections. Clean, minimal design. White background, no Ballrs branding needed — this is a utility tool.

### Section 1: Upload

```
┌─────────────────────────────────────────┐
│                                         │
│     Drag & drop screen recording here   │
│         or click to browse              │
│                                         │
│         Accepts MP4, MOV                │
│                                         │
└─────────────────────────────────────────┘
```

- Drag-and-drop zone with dashed border
- Click to open file picker
- Accept `.mp4` and `.mov` files only
- After upload, show filename and file size
- Show a small video preview (HTML5 `<video>` element, muted, controls)

### Section 2: Settings

Appears after a video is uploaded.

**Background Color**
- Text input for hex code (with `#` prefix)
- Color preview swatch next to the input (small square showing the color)
- Preset color buttons (single row, clickable pills):
  - Cream `#F5F2EB`
  - Teal `#1ABC9C`
  - Yellow `#F2C94C`
  - Black `#1A1A1A`
  - NBA `#E07A3D`
  - EPL `#A17FFF`
  - NFL `#3BA978`
  - MLB `#7A93D2`
- Clicking a preset fills the hex input and updates the swatch

**Speed**
- Radio buttons or segmented control: `1x` | `1.5x` | `2x`
- Default: `1x`

**End Card (Optional)**
- Separate upload zone (smaller than main one): "Add end card (optional)"
- Accept `.mp4` only
- After upload, show filename
- Small "Remove" button to clear it
- Note: "End card will be scaled to match and appended to the end of the video"

### Section 3: Process

- Single button: **"Create Video"**
- Disabled until a screen recording is uploaded and a background color is set
- On click:
  1. Button changes to "Processing..." with a spinner
  2. Show a progress bar based on frames processed vs total frames
  3. Show text: "Processing frame X of Y"
  4. When done, button changes to **"Download Video"**
- Download button triggers browser download of the final video file
- After download, show a "Create Another" link to reset the form

---

## Processing Pipeline — Canvas API

This is the core logic. It replicates the bash script's compositing using Canvas API for rendering and MediaRecorder API for encoding.

### Assets (bundled in the same directory)

These files must be in the same directory as `screenytime.html` and loaded as `Image` objects:

1. **`device-frame.png`** — iPhone 16 Pro device frame (1310x2710 pixels). Screen area is transparent (cut out via boolean subtract).

2. **`cover-up.png`** — Small rectangle (124x58 pixels, fill color #F5F2EB) that covers the iOS screen recording indicator (red pill).

### Layout Constants

```javascript
const FRAME_W = 1310;  // device-frame.png width
const FRAME_H = 2710;  // device-frame.png height
const SCREEN_W_PERCENT = 93;
const SCREEN_H_PERCENT = 98;
const CORNER_RADIUS = 120;
const PADDING = 80;

// Screen area (where the video goes inside the frame)
let SCREEN_W = Math.floor(FRAME_W * SCREEN_W_PERCENT / 100);
let SCREEN_H = Math.floor(FRAME_H * SCREEN_H_PERCENT / 100);
SCREEN_W = Math.floor(SCREEN_W / 2) * 2;  // ensure even
SCREEN_H = Math.floor(SCREEN_H / 2) * 2;  // ensure even

// Canvas (full output size including padding)
const CANVAS_W = Math.floor((FRAME_W + PADDING * 2) / 2) * 2;
const CANVAS_H = Math.floor((FRAME_H + PADDING * 2) / 2) * 2;

// Positioning
const SCREEN_X = Math.floor((FRAME_W - SCREEN_W) / 2) + PADDING;
const SCREEN_Y = Math.floor((FRAME_H - SCREEN_H) / 2) + PADDING;
const FRAME_X = PADDING;
const FRAME_Y = PADDING;

// Cover-up: scale to 1.53x original size, position offset
const COVERUP_SCALE = 1.53;
const COVERUP_X = SCREEN_X + 80;
const COVERUP_Y = SCREEN_Y + 40;
```

### Rendering Pipeline (per frame)

For each video frame, draw onto the canvas in this order:

```javascript
function renderFrame(ctx, videoElement, deviceFrameImg, coverUpImg, bgColor) {
  const canvas = ctx.canvas;

  // 1. Fill entire canvas with background color
  ctx.fillStyle = bgColor;
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  // 2. Draw the video frame with rounded corners
  ctx.save();

  // Create rounded rectangle clipping path at the screen position
  const x = SCREEN_X;
  const y = SCREEN_Y;
  const w = SCREEN_W;
  const h = SCREEN_H;
  const r = CORNER_RADIUS;

  ctx.beginPath();
  ctx.moveTo(x + r, y);
  ctx.lineTo(x + w - r, y);
  ctx.quadraticCurveTo(x + w, y, x + w, y + r);
  ctx.lineTo(x + w, y + h - r);
  ctx.quadraticCurveTo(x + w, y + h, x + w - r, y + h);
  ctx.lineTo(x + r, y + h);
  ctx.quadraticCurveTo(x, y + h, x, y + h - r);
  ctx.lineTo(x, y + r);
  ctx.quadraticCurveTo(x, y, x + r, y);
  ctx.closePath();
  ctx.clip();

  // Draw the video frame, scaled to fill the screen area
  // Use object-fit "cover" logic: scale to fill, center, clip overflow
  const videoAspect = videoElement.videoWidth / videoElement.videoHeight;
  const screenAspect = SCREEN_W / SCREEN_H;

  let drawW, drawH, drawX, drawY;
  if (videoAspect > screenAspect) {
    // Video is wider — fit height, crop sides
    drawH = SCREEN_H;
    drawW = SCREEN_H * videoAspect;
    drawX = SCREEN_X + (SCREEN_W - drawW) / 2;
    drawY = SCREEN_Y;
  } else {
    // Video is taller — fit width, crop top/bottom
    drawW = SCREEN_W;
    drawH = SCREEN_W / videoAspect;
    drawX = SCREEN_X;
    drawY = SCREEN_Y + (SCREEN_H - drawH) / 2;
  }

  ctx.drawImage(videoElement, drawX, drawY, drawW, drawH);
  ctx.restore();

  // 3. Draw the device frame on top (transparent screen area shows video through)
  ctx.drawImage(deviceFrameImg, FRAME_X, FRAME_Y, FRAME_W, FRAME_H);

  // 4. Draw the cover-up overlay (hides iOS recording indicator)
  const coverW = coverUpImg.naturalWidth * COVERUP_SCALE;
  const coverH = coverUpImg.naturalHeight * COVERUP_SCALE;
  ctx.drawImage(coverUpImg, COVERUP_X, COVERUP_Y, coverW, coverH);
}
```

### Main Processing Loop

```javascript
async function processVideo(videoFile, bgColor, speed, endCardFile) {
  // 1. Create offscreen canvas
  const canvas = document.createElement('canvas');
  canvas.width = CANVAS_W;
  canvas.height = CANVAS_H;
  const ctx = canvas.getContext('2d');

  // 2. Load assets
  const deviceFrame = await loadImage('device-frame.png');
  const coverUp = await loadImage('cover-up.png');

  // 3. Set up video element
  const video = document.createElement('video');
  video.src = URL.createObjectURL(videoFile);
  video.muted = true;
  video.playbackRate = speed;  // 1, 1.5, or 2
  await waitForVideoReady(video);

  const fps = 30;
  const totalFrames = Math.ceil((video.duration / speed) * fps);

  // 4. Set up MediaRecorder
  const stream = canvas.captureStream(fps);
  const mediaRecorder = new MediaRecorder(stream, getMediaRecorderOptions());

  const chunks = [];
  mediaRecorder.ondataavailable = (e) => chunks.push(e.data);

  // 5. Frame-by-frame rendering
  mediaRecorder.start();

  video.currentTime = 0;
  await video.play();

  // Use requestAnimationFrame for rendering loop
  let frameCount = 0;

  function drawLoop() {
    if (video.ended || video.paused) {
      // Video portion done — handle end card or finish
      if (endCardFile) {
        renderEndCard(endCardFile, canvas, ctx, bgColor, mediaRecorder, chunks);
      } else {
        mediaRecorder.stop();
      }
      return;
    }

    renderFrame(ctx, video, deviceFrame, coverUp, bgColor);
    frameCount++;

    // Update progress: frameCount / totalFrames
    updateProgress(frameCount, totalFrames);

    requestAnimationFrame(drawLoop);
  }

  drawLoop();

  // 6. Wait for recording to finish
  await new Promise(resolve => {
    mediaRecorder.onstop = resolve;
  });

  // 7. Return final blob
  const mimeType = mediaRecorder.mimeType || 'video/webm';
  return new Blob(chunks, { type: mimeType });
}
```

### End Card Processing

Keep the MediaRecorder running through both the main video and the end card. After the main video finishes playing, immediately start playing the end card into the same canvas. This avoids needing to concatenate two blobs.

```javascript
async function renderEndCard(endCardFile, canvas, ctx, bgColor, mediaRecorder, chunks) {
  const endCardVideo = document.createElement('video');
  endCardVideo.src = URL.createObjectURL(endCardFile);
  endCardVideo.muted = true;
  await waitForVideoReady(endCardVideo);
  await endCardVideo.play();

  function drawEndCardLoop() {
    if (endCardVideo.ended || endCardVideo.paused) {
      mediaRecorder.stop();
      return;
    }

    // Fill background
    ctx.fillStyle = bgColor;
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    // Scale end card to fit canvas (maintain aspect ratio, center, pad with bg color)
    const ecAspect = endCardVideo.videoWidth / endCardVideo.videoHeight;
    const canvasAspect = canvas.width / canvas.height;
    let drawW, drawH, drawX, drawY;

    if (ecAspect > canvasAspect) {
      drawW = canvas.width;
      drawH = canvas.width / ecAspect;
      drawX = 0;
      drawY = (canvas.height - drawH) / 2;
    } else {
      drawH = canvas.height;
      drawW = canvas.height * ecAspect;
      drawX = (canvas.width - drawW) / 2;
      drawY = 0;
    }

    ctx.drawImage(endCardVideo, drawX, drawY, drawW, drawH);
    requestAnimationFrame(drawEndCardLoop);
  }

  drawEndCardLoop();
}
```

### Helper Functions

```javascript
function loadImage(src) {
  return new Promise((resolve, reject) => {
    const img = new Image();
    img.onload = () => resolve(img);
    img.onerror = reject;
    img.src = src;
  });
}

function waitForVideoReady(video) {
  return new Promise((resolve) => {
    video.onloadeddata = resolve;
    video.load();
  });
}
```

---

## Output Scaling — Two-Canvas Approach

The final video output is **1080×1920** (standard Reels/TikTok/Shorts resolution). Internally, compositing happens at full device-frame resolution (~1470×2870) for pixel-perfect quality, then each frame is scaled down to 1080×1920 for the MediaRecorder output.

**How it works:**

1. **Render canvas** (`CANVAS_W × CANVAS_H`, ~1470×2870) — all compositing happens here at full resolution: background fill, video with rounded corners, device frame overlay, cover-up. This ensures crisp edges and accurate positioning.

2. **Output canvas** (`1080 × 1920`) — after each frame is rendered on the internal canvas, it's drawn scaled-down onto the output canvas using `drawImage`. The MediaRecorder captures its stream from this output canvas.

```javascript
const OUTPUT_W = 1080;
const OUTPUT_H = 1920;

// Per frame:
renderFrame(renderCtx, video, deviceFrame, coverUp, bgColor);    // full-res compositing
outputCtx.drawImage(renderCanvas, 0, 0, OUTPUT_W, OUTPUT_H);     // scale to 1080×1920
```

This gives the best of both worlds: full-resolution compositing (no aliasing on rounded corners or frame edges) with a Reels-ready output size. The browser's built-in bilinear/bicubic downscaling produces clean results.

---

## Output Format

- **Preferred:** `video/mp4` if the browser supports it in MediaRecorder (Chrome 131+ does)
- **Fallback:** `video/webm;codecs=vp9` (Chrome, Edge, Firefox)
- **Bitrate:** 8 Mbps minimum for high quality at this resolution
- Check `MediaRecorder.isTypeSupported()` to pick the best available format
- For social media, both MP4 and WebM work. Platforms re-encode everything anyway.

```javascript
function getMediaRecorderOptions() {
  const types = [
    'video/mp4;codecs=avc1',
    'video/mp4',
    'video/webm;codecs=vp9',
    'video/webm;codecs=vp8',
    'video/webm',
  ];
  for (const type of types) {
    if (MediaRecorder.isTypeSupported(type)) {
      return { mimeType: type, videoBitsPerSecond: 8000000 };
    }
  }
  return { videoBitsPerSecond: 8000000 };  // let browser decide
}
```

---

## Performance Expectations

The Canvas API approach is significantly faster than FFmpeg.wasm because:
- Rounded corners use GPU-accelerated clip paths (not pixel-by-pixel math)
- Canvas drawing is hardware-accelerated
- No WebAssembly overhead

**Expected processing times (rough estimates):**
- 10-second video at 1x: ~15-20 seconds
- 20-second video at 1x: ~30-40 seconds
- 20-second video at 2x: ~15-20 seconds (half the frames)

These will vary by machine. Much faster than the 5-10 minutes FFmpeg.wasm would take.

---

## File Handling

- All files stay in the browser. Nothing is uploaded to any server.
- Use the File API to read uploaded videos.
- Max file size: no hard limit, but warn if over 200MB ("Large files may take a long time to process").
- Clean up blob URLs and object URLs after download to free memory.

---

## Error Handling

| Error | Message |
|-------|---------|
| No video uploaded | "Upload a screen recording to get started" |
| Invalid file type | "Only MP4 and MOV files are supported" |
| MediaRecorder not supported | "Your browser doesn't support video recording. Try Chrome or Edge." |
| Processing fails | "Something went wrong during processing. Try a different video or refresh." |
| Video too large | "This file is very large and may take a while to process." (warning, not blocking) |

---

## Design Notes

- This is an internal utility, not a public product. Keep it simple.
- No authentication needed.
- No Ballrs branding — plain white page with a "Screenytime" title at top.
- Mobile-friendly is nice-to-have but not required. Desktop is the primary use case.
- Dark text on white background. Standard form styling.
- Preset color buttons should show the actual color as the button background with appropriate text contrast (white text on dark colors, black text on light colors).

---

## Browser Support

- **Chrome/Edge:** Full support (Canvas, MediaRecorder with MP4 or WebM)
- **Firefox:** Works with WebM output
- **Safari:** MediaRecorder support is limited. Show a message: "For best results, use Chrome or Edge."

---

## What Success Looks Like

1. User goes to ballrs.net/screenytime
2. Drags a screen recording onto the page
3. Clicks "EPL" color preset (or types a hex code)
4. Sets speed to 2x
5. Optionally uploads an end card
6. Clicks "Create Video"
7. Sees progress bar advancing, takes under a minute
8. Downloads the final video
9. Result looks identical to running the bash script locally

---

## Files Needed

Along with this PRD, provide these asset files:

1. `device-frame.png` — from `~/Projects/screenytime/device-frame.png`
2. `cover-up.png` — from `~/Projects/screenytime/cover-up.png`

Both go in the root of `~/Projects/ballrs-website/` alongside `screenytime.html`.

---

## Out of Scope

- Audio from the screen recording (social content is typically muted or has music added after)
- Multiple device frames (just iPhone 16 Pro for now)
- Batch processing multiple videos
- User accounts or saving history
- Custom cover-up positioning (hardcoded values from the bash script)
- Server-side processing
