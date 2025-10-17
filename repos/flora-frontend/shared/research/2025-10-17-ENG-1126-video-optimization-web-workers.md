---
date: 2025-10-17 15:53:50 EDT
researcher: matanshavit
git_commit: 1d413c2fc2a94dd3a6b4ca0839d5a148ce76984f
branch: eng-1126
repository: eng-1126
topic: "Preventing Web Worker Timeouts for Video Processing through ImageKit Optimizations and Streaming Strategies"
tags: [research, codebase, video-processing, web-workers, imagekit, webgl, streaming, r3f-canvas, timeouts, optimization]
status: complete
last_updated: 2025-10-17
last_updated_by: matanshavit
---

# Research: Preventing Web Worker Timeouts for Video Processing through ImageKit Optimizations and Streaming Strategies

**Date**: 2025-10-17 15:53:50 EDT
**Researcher**: matanshavit
**Git Commit**: 1d413c2fc2a94dd3a6b4ca0839d5a148ce76984f
**Branch**: eng-1126
**Repository**: eng-1126

## Research Question

Now that we finished implementing for ticket ENG-1057, we want to make sure the web workers do not time out, as much as possible. Research:
- Getting the webm version of mp4 files from ImageKit instead of the full size file
- How to convert the resolution lower or cap it at a certain limit
- How to truncate the play time to a certain limit
- Investigate the possibility of streaming the file, even though it still has to be rendered onto a WebGL texture
- Whether we can put the timeout on the actual web call instead of the worker's running time
- Any other relevant strategies to prevent web worker failures

## Summary

The current video processing system in the r3F canvas uses a Web Worker to download and process videos off the main thread, with a 15-second timeout limit. This research reveals multiple optimization strategies available through ImageKit's video transformation API and browser streaming APIs to prevent timeout failures. Key findings include: ImageKit supports format conversion (MP4→WebM), resolution capping (e.g., 720p), duration trimming (e.g., 10 seconds), and adaptive bitrate streaming (HLS/DASH) - all through URL parameters. Additionally, the browser's Media Source Extensions API enables true streaming to WebGL textures without loading entire files, and the timeout mechanism can be adjusted to focus on network operations rather than total worker runtime.

## Detailed Findings

### Current Video Processing Architecture

#### Web Worker Video Loading System

The video processing system centers around `VideoTextureCoordinator` (`src/lib/videos/video-texture-loader.ts:83-648`) which orchestrates loading through a Web Worker:

**Worker Initialization** (`video-texture-loader.ts:97-217`):
- Preloads worker file for caching
- 3 retry attempts with exponential backoff (1000ms base delay)
- 5-second ping timeout to verify worker responsiveness
- Falls back to main thread after all retries fail

**Worker Processing Flow** (`video-processor.worker.js:77-127`):
1. Sends HEAD request for content-type and content-length
2. Posts video metadata to main thread
3. Downloads full video file with `fetch(videoUrl)`
4. Converts response to blob
5. Creates object URL from blob
6. Caches in `videoCache` by blockId
7. Returns object URL to main thread

**The Critical 15-Second Timeout** (`video-texture-loader.ts:396-409`):
- Hard-coded timeout of 15000ms for entire worker operation
- Includes full video download, blob conversion, and object URL creation
- Timeout encompasses network latency, download time, and processing
- On timeout, deletes callback and rejects with detailed error

**Main Thread Fallback** (`video-texture-loader.ts:513-587`):
- Creates video element directly on main thread
- Uses `requestIdleCallback` to prevent UI blocking (2000ms timeout)
- Safari fallback uses direct loading without idle callback

### ImageKit Video Transformation Capabilities

ImageKit provides comprehensive video optimization through URL parameters, all using the `?tr=<parameters>` syntax.

#### 1. Format Conversion (MP4 to WebM)

**Parameters**:
- `f-webm`: Force WebM output with VP9 codec
- `f-mp4`: Force MP4 output with H.264 codec
- `f-auto`: Automatic format selection based on browser support
- `vc-vp9`: Use VP9 video codec (WebM)
- `vc-av1`: Use AV1 codec (best compression)
- `ac-opus`: Use Opus audio codec (WebM)

**Example Implementation**:
```javascript
// In video-texture-loader.ts loadVideoTexture method
const optimizedUrl = `${videoUrl}?tr=f-webm,vc-vp9,ac-opus`;
```

**Benefits**: WebM with VP9 provides ~30% smaller file sizes than H.264 MP4

#### 2. Resolution Capping (720p Limit)

**Parameters**:
- `h-720`: Cap height at 720px
- `w-1280`: Cap width at 1280px
- `c-at_max`: Limit to maximum dimensions without exceeding
- `ar-16_9`: Maintain specific aspect ratio

**Example Implementation**:
```javascript
// Cap at 720p while maintaining aspect ratio
const cappedUrl = `${videoUrl}?tr=h-720,c-at_max`;

// Or specify exact 720p dimensions
const exact720pUrl = `${videoUrl}?tr=w-1280,h-720,c-at_max`;
```

**Benefits**: 720p reduces file size by ~60-75% compared to 1080p

#### 3. Duration Truncation (10 Second Limit)

**Parameters**:
- `du-10`: Output will be exactly 10 seconds
- `eo-10`: Keep only first 10 seconds
- `so-5,du-10`: Extract 10 seconds starting at 5-second mark

**Example Implementation**:
```javascript
// Limit to first 10 seconds
const truncatedUrl = `${videoUrl}?tr=du-10`;

// Extract middle portion
const segmentUrl = `${videoUrl}?tr=so-5,du-10`;
```

**Benefits**: Linear reduction in file size and download time

#### 4. Combined Optimization Strategy

**Recommended transformation chain for maximum optimization**:
```javascript
function getOptimizedVideoUrl(originalUrl: string, options = {}) {
  const {
    format = 'webm',
    maxHeight = 720,
    maxDuration = 10,
    quality = 50
  } = options;

  const transformations = [
    `f-${format}`,
    `h-${maxHeight}`,
    `c-at_max`,
    `du-${maxDuration}`,
    `q-${quality}`
  ];

  return `${originalUrl}?tr=${transformations.join(',')}`;
}

// Usage in video-texture-loader.ts
const optimizedUrl = getOptimizedVideoUrl(videoUrl, {
  format: 'webm',
  maxHeight: 720,
  maxDuration: 10,
  quality: 50
});
```

**Expected Results**: Combined optimizations can reduce file size by 80-90%

### Current ImageKit Usage Patterns

The codebase already implements several ImageKit patterns that can be adapted for video optimization:

#### URL Transformation Patterns

**Query Parameter Transformations** (`src/lib/utils.ts:233-242`):
```typescript
export function toHumaneSizedImageUrl(url: string, size: { width: number; height: number }) {
  if (url.startsWith("https://ik.imagekit.io/")) {
    return `${url}?tr=w-${size.width},h-${size.height}`;
  }
  return url;
}
```

**Path-Based Transformations** (`src/lib/images/helpers.ts:368-390`):
- Inserts transformation segment in URL path
- Pattern: `/account/tr:w-1536/path/to/file.jpg`
- Currently used for image downscaling

**Video Thumbnail Generation** (`src/lib/videos/helpers.ts:196-214`):
```typescript
export function getVideoThumbnailUrl(videoUrl: string | undefined): string | undefined {
  if (!videoUrl?.startsWith("https://ik.imagekit.io/")) return undefined;
  const cleanUrl = videoUrl.split("?")[0];
  return `${cleanUrl}/ik-thumbnail.jpg`;
}
```

#### Avoiding 302 Redirects

**Original Video Flag** (`src/lib/schema/map-before-request.ts:1435-1440`):
```typescript
// Avoid 302 response on first transformation
const parts = newParams.video_url.split("/");
if (!parts.includes("tr:orig")) {
  newParams.video_url = [...parts.slice(0, 4), "tr:orig", ...parts.slice(4)].join("/");
}
```

**Note**: This pattern bypasses transformations - opposite of what we want for optimization

### Video Streaming Implementation for WebGL Textures

#### Browser Streaming APIs

**Media Source Extensions (MSE)**:
- Provides fine-grained control over video buffering
- Enables progressive loading without downloading entire file
- Supported in Web Workers (Chrome 108+)
- Works with standard `<video>` elements through `srcObject`

**Implementation Pattern**:
```javascript
// In video-processor.worker.js - replace current blob download
async function streamVideo(videoUrl, blockId) {
  const mediaSource = new MediaSource();
  const objectUrl = URL.createObjectURL(mediaSource);

  mediaSource.addEventListener('sourceopen', async () => {
    const sourceBuffer = mediaSource.addSourceBuffer('video/webm; codecs="vp9,opus"');
    const response = await fetch(videoUrl);
    const reader = response.body.getReader();

    while (true) {
      const { value, done } = await reader.read();
      if (done) break;

      await new Promise(resolve => {
        sourceBuffer.appendBuffer(value);
        sourceBuffer.onupdateend = resolve;
      });

      // Post progress updates to main thread
      self.postMessage({
        type: 'videoProgress',
        blockId,
        loaded: sourceBuffer.buffered.length > 0
          ? sourceBuffer.buffered.end(0) : 0
      });
    }
  });

  return objectUrl;
}
```

**THREE.VideoTexture Integration**:
- VideoTexture automatically updates from streaming video element
- No changes needed to existing `VideoMaterial` component
- Browser handles buffering and frame availability

#### Adaptive Bitrate Streaming via ImageKit

**HLS Implementation**:
```javascript
// Request HLS manifest from ImageKit
const hlsManifestUrl = `${videoUrl}/ik-master.m3u8?tr=sr-240_360_480_720`;

// Use HLS.js library for browsers without native support
import Hls from 'hls.js';

if (Hls.isSupported()) {
  const hls = new Hls();
  hls.loadSource(hlsManifestUrl);
  hls.attachMedia(video);
  hls.on(Hls.Events.MANIFEST_PARSED, () => {
    // Video ready for texture creation
    const texture = new THREE.VideoTexture(video);
  });
}
```

**Benefits**:
- Starts playback with lowest quality (240p) for fast initial load
- Automatically adapts to network conditions
- Prevents timeout by loading incrementally

### Timeout Management Strategies

#### Current Timeout Issue

The 15-second timeout (`video-texture-loader.ts:409`) encompasses:
1. Network request initiation
2. Full file download
3. Blob conversion in memory
4. Object URL creation

For large videos, download alone can exceed 15 seconds.

#### Proposed Solutions

**1. Separate Network Timeout from Processing Timeout**:
```javascript
// Modified loadInWorker method
private async loadInWorker(videoUrl: string, blockId: string, options: VideoLoadOptions) {
  return new Promise((resolve, reject) => {
    const networkTimeout = 30000; // 30s for download
    const processingTimeout = 5000; // 5s for processing
    let downloadComplete = false;

    const networkTimer = setTimeout(() => {
      if (!downloadComplete) {
        reject(new Error('Network timeout: Download exceeded 30 seconds'));
      }
    }, networkTimeout);

    const callback = (message: WorkerMessage) => {
      if (message.type === 'downloadComplete') {
        downloadComplete = true;
        clearTimeout(networkTimer);

        // Start processing timeout
        setTimeout(() => {
          reject(new Error('Processing timeout: Video processing exceeded 5 seconds'));
        }, processingTimeout);
      } else if (message.type === 'videoReady') {
        // Success - clear all timeouts and resolve
        resolve(createVideoTexture(message.data));
      }
    };

    this.workerCallbacks.set(blockId, callback);
    this.worker.postMessage({ type: 'loadVideo', videoUrl, blockId });
  });
}
```

**2. Progressive Loading with Early Texture Creation**:
```javascript
// Create texture immediately, update as data arrives
private async loadProgressively(videoUrl: string, blockId: string) {
  const video = document.createElement('video');
  video.src = videoUrl;
  video.preload = 'metadata';

  // Create texture immediately with first frame
  await new Promise(resolve => {
    video.addEventListener('loadeddata', resolve, { once: true });
  });

  const texture = new THREE.VideoTexture(video);

  // Continue loading in background
  video.load();

  return { texture, video, partial: true };
}
```

**3. Implement Chunked Download in Worker**:
```javascript
// In video-processor.worker.js
async function downloadInChunks(url, chunkSize = 1024 * 1024) { // 1MB chunks
  const chunks = [];
  let downloaded = 0;

  // Get total size
  const headResponse = await fetch(url, { method: 'HEAD' });
  const totalSize = parseInt(headResponse.headers.get('content-length'));

  while (downloaded < totalSize) {
    const start = downloaded;
    const end = Math.min(downloaded + chunkSize - 1, totalSize - 1);

    const response = await fetch(url, {
      headers: { 'Range': `bytes=${start}-${end}` }
    });

    chunks.push(await response.arrayBuffer());
    downloaded = end + 1;

    // Report progress
    self.postMessage({
      type: 'downloadProgress',
      progress: downloaded / totalSize
    });
  }

  // Combine chunks into blob
  return new Blob(chunks);
}
```

### Historical Context from Past Research

#### Previous Findings on Web Worker Timeouts

From `thoughts/shared/research/2025-10-14-ENG-1057-webworker-timeout-root-causes.md`:

**Root Causes Identified**:
1. Blocking network operations - Full video download required
2. Memory-intensive blob creation for large videos
3. Sequential processing without streaming
4. Network bandwidth constraints vs 15-second timeout
5. Worker overhead without streaming benefits

**Key Insight**: "The worker provides no benefit over main thread for simple download-and-create-blob operations. The benefit would come from implementing streaming or chunked processing."

#### ImageKit Video Thumbnail Implementation

From `thoughts/shared/research/2025-10-16-ENG-1057-video-thumbnails-imagekit.md`:

**Already Implemented Pattern**:
- ImageKit thumbnail generation working for images
- Video thumbnail URL pattern: `${videoUrl}/ik-thumbnail.jpg`
- Can specify time offset: `tr=so-<seconds>` for specific frames

This existing pattern can be extended for preview generation during loading.

### Comprehensive Optimization Strategy

Based on all findings, here's the recommended implementation approach:

#### Phase 1: Quick Wins (Immediate Implementation)

1. **Add ImageKit URL Transformations**:
```javascript
// In video-texture-loader.ts
private getOptimizedUrl(originalUrl: string, quality: 'low' | 'medium' | 'high'): string {
  if (!originalUrl.startsWith('https://ik.imagekit.io/')) {
    return originalUrl;
  }

  const qualitySettings = {
    low: { height: 480, quality: 40, duration: 30 },
    medium: { height: 720, quality: 50, duration: 60 },
    high: { height: 1080, quality: 60, duration: null }
  };

  const settings = qualitySettings[quality];
  const params = [
    'f-webm',  // Use WebM for better compression
    `h-${settings.height}`,
    'c-at_max',
    `q-${settings.quality}`
  ];

  if (settings.duration) {
    params.push(`du-${settings.duration}`);
  }

  return `${originalUrl}?tr=${params.join(',')}`;
}
```

2. **Separate Network and Processing Timeouts**:
- Increase network timeout to 30 seconds
- Keep processing timeout at 5 seconds
- Add progress reporting from worker

#### Phase 2: Streaming Implementation

1. **Implement MSE in Worker**:
- Replace blob download with streaming approach
- Progressive buffer management
- Early object URL creation

2. **Add HLS.js Support**:
- Detect HLS support
- Fallback to progressive download
- Integrate with VideoMaterial component

#### Phase 3: Advanced Optimizations

1. **Predictive Preloading**:
- Preload video metadata on hover
- Cache thumbnails aggressively
- Prefetch based on viewport proximity

2. **Quality Adaptation**:
- Start with low quality
- Upgrade quality after initial load
- Base quality on device capabilities

## Code References

### Core Video Processing
- `src/lib/videos/video-texture-loader.ts:83-648` - VideoTextureCoordinator class
- `public/workers/video-processor.worker.js:77-127` - Worker video processing
- `src/components/r3f/blocks/video/video-material.tsx:96-182` - Video material loading

### ImageKit Integration
- `src/lib/imagekit/helpers.ts:7-20` - ImageKit client initialization
- `src/lib/videos/helpers.ts:196-214` - Video thumbnail URL generation
- `src/lib/utils.ts:233-242` - Image URL transformation helper

### Timeout Configuration
- `src/lib/videos/video-texture-loader.ts:409` - 15-second timeout definition
- `src/lib/videos/video-texture-loader.ts:91-93` - Worker configuration constants

## Architecture Documentation

The video processing system follows this flow:

1. **VideoMaterial** requests texture from **VideoTextureCoordinator**
2. Coordinator attempts **Web Worker** loading first
3. Worker downloads full video and creates blob URL
4. On success, returns THREE.VideoTexture to VideoMaterial
5. On timeout/failure, falls back to main thread loading
6. VideoMaterial renders with VideoShaderMaterial

Key architectural constraint: Videos must be fully downloaded before texture creation in current implementation.

## Historical Context (from thoughts/)

- `thoughts/shared/research/2025-10-14-ENG-1057-webworker-timeout-root-causes.md` - Comprehensive analysis of timeout causes
- `thoughts/shared/plans/2025-01-14-ENG-1057-video-lazy-loading.md` - Lazy loading implementation (completed)
- `thoughts/shared/research/2025-10-16-ENG-1057-video-thumbnails-imagekit.md` - ImageKit thumbnail capability analysis
- `thoughts/shared/research/2025-10-14-ENG-1057-video-loading-r3f-canvas.md` - Complete video loading architecture documentation

## Related Research

- [2025-10-14-ENG-1057-video-loading-r3f-canvas.md](thoughts/shared/research/2025-10-14-ENG-1057-video-loading-r3f-canvas.md) - Video loading architecture in r3F
- [2025-10-14-ENG-1057-webworker-timeout-root-causes.md](thoughts/shared/research/2025-10-14-ENG-1057-webworker-timeout-root-causes.md) - Root cause analysis of timeouts
- [2025-10-16-ENG-1057-video-thumbnails-imagekit.md](thoughts/shared/research/2025-10-16-ENG-1057-video-thumbnails-imagekit.md) - ImageKit thumbnail capabilities

## Open Questions

1. **Format Compatibility**: Should we use WebM universally or implement browser detection for format selection?
2. **Quality Presets**: What should be the default quality settings for different device types?
3. **Fallback Strategy**: If optimized URL fails, should we retry with original or fail fast?
4. **Caching Policy**: How long should transformed videos be cached in the worker?
5. **Progress UI**: Should we expose download progress to the user interface?

## Implementation Recommendations

### Immediate Actions (Can implement now)

1. **Add ImageKit transformations** to `VideoTextureCoordinator.loadVideoTexture()`:
   - Default to 720p, 30 seconds, WebM format
   - Make configurable via VideoLoadOptions

2. **Increase network timeout** to 30 seconds:
   - Separate from processing timeout
   - Add timeout reason to error messages

3. **Implement quality levels** based on existing pattern:
   - Low: 480p, 40% quality, 30s max
   - Medium: 720p, 50% quality, 60s max
   - High: Original resolution, 60% quality

### Future Enhancements

1. **Streaming via MSE**: Requires worker refactoring but eliminates timeout issues
2. **HLS.js Integration**: For adaptive bitrate streaming
3. **Intelligent Prefetching**: Based on viewport and user interaction patterns
4. **WebCodecs API**: When browser support improves