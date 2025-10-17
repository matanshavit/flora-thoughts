# ImageKit Video Optimization Implementation Plan

## Overview

Add ImageKit transformations to optimize video loading in the r3F canvas, preventing Web Worker timeouts by reducing file sizes through format conversion, resolution capping, and duration limiting. This is a targeted optimization that leverages existing ImageKit infrastructure.

## Current State Analysis

The video loading system currently downloads full-size videos through a Web Worker with a 15-second timeout. Based on research, videos frequently exceed this timeout due to large file sizes. ImageKit supports video transformations via URL parameters that can reduce file sizes by 80-90%.

### Key Discoveries:
- VideoTextureCoordinator (`src/lib/videos/video-texture-loader.ts:338-382`) accepts VideoLoadOptions with quality field
- Worker receives options object (`public/workers/video-processor.worker.js:77`) but doesn't use it for URL transformation
- ImageKit transformation patterns exist (`src/lib/utils.ts:233-242` for images, `src/lib/schema/map-before-request.ts:1435-1440` for videos)
- Query parameter format (`?tr=`) is simpler than path-based transformations

## Desired End State

Videos load reliably within the 15-second timeout by automatically applying ImageKit optimizations based on quality settings. The system should:
- Apply WebM format conversion, resolution capping (720p), and duration limits (10s) by default
- Make transformations configurable via VideoLoadOptions
- Work only for ImageKit URLs (non-breaking for other sources)

### Success Verification:
- Large videos that previously timed out now load successfully
- Video URLs contain appropriate transformation parameters
- Non-ImageKit videos continue working unchanged

## What We're NOT Doing

- Implementing streaming or chunked downloads (more complex)
- Modifying timeout duration (keeping 15 seconds)
- Changing the Web Worker architecture
- Adding HLS/adaptive bitrate streaming
- Implementing progressive texture loading
- Creating new UI for quality selection

## Implementation Approach

Use the simplest approach: add URL transformations in the VideoTextureCoordinator before passing URLs to the worker. This requires minimal changes and leverages existing ImageKit infrastructure.

## Phase 1: Add ImageKit URL Transformation Helper

### Overview

Create a helper function to apply ImageKit video transformations based on quality settings.

### Changes Required:

#### 1. Video Texture Loader Enhancement

**File**: `src/lib/videos/video-texture-loader.ts`
**Changes**: Add helper method and modify URL before worker/main thread loading

```typescript
// After line 83, add new helper method in VideoTextureCoordinator class:

/**
 * Apply ImageKit video optimizations based on quality settings
 * @param videoUrl - Original video URL
 * @param quality - Quality preset (low/medium/high)
 * @returns Optimized URL with ImageKit transformations
 */
private getOptimizedVideoUrl(
  videoUrl: string,
  quality: "low" | "medium" | "high" = "medium"
): string {
  // Only process ImageKit URLs
  if (!videoUrl.startsWith("https://ik.imagekit.io/")) {
    return videoUrl;
  }

  // Skip if already has transformations to avoid double-processing
  if (videoUrl.includes("?tr=") || videoUrl.includes("/tr:")) {
    return videoUrl;
  }

  const qualitySettings = {
    low: {
      format: "webm",
      height: 480,
      quality: 40,
      duration: 10
    },
    medium: {
      format: "webm",
      height: 720,
      quality: 50,
      duration: 10
    },
    high: {
      format: "webm",
      height: 1080,
      quality: 60,
      duration: 10
    }
  };

  const settings = qualitySettings[quality];
  const transformations = [
    `f-${settings.format}`,     // Format conversion to WebM
    `h-${settings.height}`,      // Height limit (maintains aspect ratio)
    `c-at_max`,                  // Constraint to not exceed dimensions
    `q-${settings.quality}`,     // Quality/compression level
    `du-${settings.duration}`    // Duration limit in seconds
  ];

  return `${videoUrl}?tr=${transformations.join(',')}`;
}

// Modify loadVideoTexture method (around line 338-382):
async loadVideoTexture(
  videoUrl: string,
  blockId: string,
  options: VideoLoadOptions = {},
): Promise<VideoLoadResult> {
  // Apply optimizations based on quality setting
  const optimizedUrl = this.getOptimizedVideoUrl(
    videoUrl,
    options.quality || "medium"
  );

  if (!this.initialized || !this.worker) {
    await this.initialize();
  }

  // ... rest of the method remains the same, but use optimizedUrl instead of videoUrl

  const loadPromise = this.worker
    ? this.loadInWorker(optimizedUrl, blockId, options)  // Pass optimizedUrl
    : this.loadInMainThread(optimizedUrl, blockId, options);  // Pass optimizedUrl
```

### Success Criteria:

#### Automated Verification:
- [x] Type checking passes: `pnpm typecheck`
- [x] Build succeeds: `pnpm build`
- [x] Existing tests pass: `pnpm test:unit`

#### Manual Verification:
- [ ] Videos load successfully with transformation parameters in URL
- [ ] Large videos that previously timed out now load within 15 seconds
- [ ] Video quality is acceptable at each quality level
- [ ] Non-ImageKit videos continue working unchanged

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 2: Make Transformations Configurable

### Overview

Extend VideoLoadOptions to allow fine-grained control over transformations.

### Changes Required:

#### 1. Extend VideoLoadOptions Interface

**File**: `src/lib/videos/video-texture-loader.ts`
**Changes**: Add transformation options to interface

```typescript
// Modify VideoLoadOptions interface (around line 19-27):
export interface VideoLoadOptions {
  loop?: boolean;
  muted?: boolean;
  autoplay?: boolean;
  quality?: "low" | "medium" | "high";
  width?: number;
  height?: number;
  // New transformation options
  maxHeight?: number;        // Override default height limit
  maxDuration?: number;      // Override default duration limit (seconds)
  videoFormat?: "webm" | "mp4" | "auto";  // Override format
  compressionQuality?: number;  // 1-100, override quality
}

// Update getOptimizedVideoUrl to use these options:
private getOptimizedVideoUrl(
  videoUrl: string,
  options: VideoLoadOptions = {}
): string {
  if (!videoUrl.startsWith("https://ik.imagekit.io/")) {
    return videoUrl;
  }

  if (videoUrl.includes("?tr=") || videoUrl.includes("/tr:")) {
    return videoUrl;
  }

  // Use custom options or fall back to quality presets
  const quality = options.quality || "medium";
  const qualitySettings = {
    low: { format: "webm", height: 480, quality: 40, duration: 10 },
    medium: { format: "webm", height: 720, quality: 50, duration: 10 },
    high: { format: "webm", height: 1080, quality: 60, duration: 10 }
  };

  const defaults = qualitySettings[quality];

  // Apply custom overrides
  const settings = {
    format: options.videoFormat || defaults.format,
    height: options.maxHeight || defaults.height,
    quality: options.compressionQuality || defaults.quality,
    duration: options.maxDuration !== undefined ? options.maxDuration : defaults.duration
  };

  const transformations = [];

  if (settings.format !== "auto") {
    transformations.push(`f-${settings.format}`);
  }
  transformations.push(`h-${settings.height}`);
  transformations.push(`c-at_max`);
  transformations.push(`q-${settings.quality}`);

  if (settings.duration > 0) {
    transformations.push(`du-${settings.duration}`);
  }

  return `${videoUrl}?tr=${transformations.join(',')}`;
}

// Update loadVideoTexture to use getOptimizedVideoUrl with full options:
async loadVideoTexture(
  videoUrl: string,
  blockId: string,
  options: VideoLoadOptions = {},
): Promise<VideoLoadResult> {
  const optimizedUrl = this.getOptimizedVideoUrl(videoUrl, options);
  // ... rest remains the same
}
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `pnpm typecheck`
- [ ] Build succeeds: `pnpm build`
- [ ] Unit tests pass: `pnpm test:unit`

#### Manual Verification:
- [ ] Custom transformation options are applied correctly
- [ ] Quality presets work when custom options not provided
- [ ] Videos with duration=0 don't get duration transformation
- [ ] Format="auto" skips format transformation

---

## Phase 3: Add Logging and Monitoring

### Overview

Add debug logging to track transformation effectiveness.

### Changes Required:

#### 1. Add Transformation Logging

**File**: `src/lib/videos/video-texture-loader.ts`
**Changes**: Add logging for transformation application

```typescript
// In getOptimizedVideoUrl method, before return:
if (!isProdEnv) {
  const wasTransformed = videoUrl !== optimizedUrl;
  if (wasTransformed) {
    console.log(`Video URL optimized:`, {
      original: videoUrl,
      optimized: optimizedUrl,
      quality: options.quality || "medium",
      transformations: transformations
    });
  }
}

// In loadVideoTexture, track if optimization was applied:
const wasOptimized = videoUrl !== optimizedUrl;
if (!isProdEnv && wasOptimized) {
  videoPerformanceMonitor.recordOptimization(blockId, {
    originalUrl: videoUrl,
    optimizedUrl: optimizedUrl,
    quality: options.quality || "medium"
  });
}
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `pnpm typecheck`
- [ ] Build succeeds: `pnpm build`
- [ ] No console logs appear in production build

#### Manual Verification:
- [ ] Console shows optimization details in development
- [ ] Performance monitor tracks optimized videos
- [ ] No logging in production environment

---

## Testing Strategy

### Unit Tests:
- Test URL transformation logic with various inputs
- Verify non-ImageKit URLs remain unchanged
- Test quality preset application
- Test custom option overrides

### Integration Tests:
- Load ImageKit video with each quality setting
- Verify timeout behavior with large videos
- Test fallback to main thread with optimized URLs

### Manual Testing Steps:
1. Load a project with ImageKit videos
2. Verify transformation parameters in Network tab
3. Test with videos that previously timed out
4. Check video quality at each preset level
5. Verify non-ImageKit videos still work

## Performance Considerations

- WebM format reduces file size by ~30% vs MP4
- 720p reduces size by ~60-75% vs 1080p
- 10-second duration cap provides predictable max file size
- First transformation request may be slower (ImageKit processing)
- Subsequent requests use ImageKit's CDN cache

## Migration Notes

- No migration needed - enhancement is backwards compatible
- Existing videos without options will use "medium" quality default
- Already-transformed URLs are detected and skipped

## References

- Original research: `thoughts/shared/research/2025-10-17-ENG-1126-video-optimization-web-workers.md`
- VideoTextureCoordinator: `src/lib/videos/video-texture-loader.ts:83-648`
- Worker implementation: `public/workers/video-processor.worker.js:77-127`
- ImageKit docs: https://imagekit.io/docs/video-transformation