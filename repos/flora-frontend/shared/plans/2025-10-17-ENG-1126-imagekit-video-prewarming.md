# ImageKit Video Pre-warming Implementation Plan

## Overview

Implement a pre-warming mechanism for ImageKit video transformations to eliminate the initial loading delay when videos are first displayed. This will trigger ImageKit to generate and cache the transformed video versions (WebM format with quality presets) immediately when video URLs become available, rather than waiting for user interaction.

## Current State Analysis

Videos in Flora go through this flow:
1. AI provider generates video → Downloads to temporary URL
2. Video uploaded to ImageKit → Permanent URL created
3. URL stored in `nodeOutputsMap[nodeId].videoUrl`
4. User hovers/selects node → VideoMaterial component loads
5. **DELAY HERE**: First request to transformed URL triggers ImageKit processing (2-5 seconds)
6. Subsequent loads are instant (cached forever by ImageKit)

The optimization system (`getOptimizedVideoUrl`) adds transformations like `?tr=f-webm,h-720,c-at_max,q-50,du-10` to convert videos to WebM format with specific quality settings.

## Desired End State

Videos should load instantly when users first interact with them. The pre-warming should:
- Trigger automatically when video URLs become available
- Run in the background without blocking UI
- Support all three quality presets (low, medium, high)
- Not interfere with normal video loading flow
- Be transparent to the user

### Success Verification:
- First video interaction loads as quickly as subsequent ones
- Network tab shows pre-warming HEAD requests to transformation URLs
- No visible loading delays when hovering over video nodes

## What We're NOT Doing

- NOT modifying the upload process to use post-transformations (would require backend changes)
- NOT implementing polling to wait for transformations (adds complexity)
- NOT changing the existing video loading architecture
- NOT pre-loading actual video content (only triggering transformation generation)
- NOT modifying the quality selection logic

## Implementation Approach

Add a minimal pre-warming method to `VideoTextureCoordinator` that sends HEAD requests to transformation URLs. Hook this into the existing flow when nodes receive video URLs, using requestIdleCallback for non-blocking execution.

## Phase 1: Add Pre-warming Method to VideoTextureCoordinator

### Overview

Add a `warmVideoTransformations` method that triggers ImageKit to generate transformed versions by making HEAD requests to the transformation URLs.

### Changes Required:

#### 1. VideoTextureCoordinator Class Enhancement

**File**: `src/lib/videos/video-texture-loader.ts`
**Changes**: Add warming method after the `getOptimizedVideoUrl` method (around line 152)

```typescript
  /**
   * Pre-warm ImageKit video transformations by triggering generation
   * @param videoUrl - Original video URL to warm
   * @param qualities - Quality presets to warm (defaults to all)
   * @returns Promise that resolves when warming requests are sent
   */
  async warmVideoTransformations(
    videoUrl: string,
    qualities: Array<"low" | "medium" | "high"> = ["low", "medium", "high"]
  ): Promise<void> {
    // Only process ImageKit URLs
    if (!videoUrl.startsWith("https://ik.imagekit.io/")) {
      return;
    }

    // Skip if URL already has transformations
    if (videoUrl.includes("?tr=") || videoUrl.includes("/tr:")) {
      return;
    }

    const warmingPromises = qualities.map(async (quality) => {
      try {
        const optimizedUrl = this.getOptimizedVideoUrl(videoUrl, quality);

        // Use HEAD request to trigger transformation without downloading
        await fetch(optimizedUrl, {
          method: "HEAD",
          cache: "no-store", // Ensure we hit ImageKit, not browser cache
          redirect: "manual" // Don't follow 302 redirects (indicates processing)
        });
      } catch {
        // Silently ignore errors - warming is best-effort
        // Errors are expected (302 redirects, CORS, etc.)
      }
    });

    // Fire and forget - don't wait for completion
    void Promise.all(warmingPromises);
  }
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compilation passes: `pnpm typecheck`
- [ ] No linting errors: `pnpm lint`
- [ ] Unit tests pass: `pnpm test:unit`

#### Manual Verification:
- [ ] Method can be called without errors
- [ ] HEAD requests appear in network tab for transformation URLs
- [ ] No impact on existing video loading functionality

---

## Phase 2: Integrate Pre-warming into Node Update Flow

### Overview

Hook the pre-warming into the existing flow when nodes receive video URLs after generation completes.

### Changes Required:

#### 1. Hook into Node Generation Completion

**File**: `src/hooks/use-node-generation.tsx`
**Changes**: Add warming call when video URL is received (after line 140)

```typescript
// After line 140, where nodeOutput is set with videoUrl
if (resultType === FloraEndpointGenerate_ResultType.VIDEO_URL && mediaUrl) {
  // Import at top of file
  // import { videoCoordinator } from "@/lib/videos/video-texture-loader";

  // Warm transformations in background using requestIdleCallback
  if ("requestIdleCallback" in window) {
    requestIdleCallback(() => {
      void videoCoordinator.warmVideoTransformations(mediaUrl);
    }, { timeout: 5000 });
  } else {
    // Fallback for Safari
    setTimeout(() => {
      void videoCoordinator.warmVideoTransformations(mediaUrl);
    }, 1000);
  }
}
```

#### 2. Add Import Statement

**File**: `src/hooks/use-node-generation.tsx`
**Changes**: Add import at the top of the file (around line 1-10)

```typescript
import { videoCoordinator } from "@/lib/videos/video-texture-loader";
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compilation passes: `pnpm typecheck`
- [ ] Build succeeds: `pnpm build`
- [ ] No circular dependency warnings

#### Manual Verification:
- [ ] Generate a new video in the app
- [ ] Observe HEAD requests to transformation URLs in network tab immediately after generation
- [ ] First hover over video node loads quickly (no 2-5 second delay)
- [ ] Subsequent hovers remain fast

---

## Phase 3: Add Pre-warming for Existing Videos on Canvas Load

### Overview

When the canvas loads with existing video nodes, pre-warm their transformations to ensure smooth interaction.

### Changes Required:

#### 1. FlowCore Component Enhancement

**File**: `src/components/flow/flow-core.tsx`
**Changes**: Add effect to warm existing video URLs on mount

```typescript
// Add this useEffect after other effects in the FlowCore component
// Around line 200-250 (after existing useEffects)

useEffect(() => {
  // Warm video transformations for all existing video nodes
  const warmExistingVideos = () => {
    const videoUrls = Object.values(nodeOutputsMap)
      .filter((output): output is VideoUrlOutput =>
        output && 'videoUrl' in output && typeof output.videoUrl === 'string'
      )
      .map(output => output.videoUrl)
      .filter(url => url.startsWith("https://ik.imagekit.io/"));

    // Warm each unique video URL
    const uniqueUrls = Array.from(new Set(videoUrls));
    uniqueUrls.forEach(url => {
      void videoCoordinator.warmVideoTransformations(url);
    });
  };

  // Use requestIdleCallback for non-blocking warming
  if ("requestIdleCallback" in window) {
    requestIdleCallback(warmExistingVideos, { timeout: 10000 });
  } else {
    setTimeout(warmExistingVideos, 2000);
  }
}, []); // Run once on mount
```

#### 2. Add Required Imports

**File**: `src/components/flow/flow-core.tsx`
**Changes**: Add imports at the top

```typescript
import { videoCoordinator } from "@/lib/videos/video-texture-loader";
import type { VideoUrlOutput } from "@/components/nodes/types";
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compilation passes: `pnpm typecheck`
- [ ] No performance regression in canvas load time
- [ ] Build succeeds: `pnpm build`

#### Manual Verification:
- [ ] Load a project with existing video nodes
- [ ] Network tab shows HEAD requests for video transformations after page load
- [ ] First interaction with any video node is instant (no delay)
- [ ] No blocking of UI during warming

---

## Testing Strategy

### Unit Tests:
- Test `warmVideoTransformations` with various URL formats
- Verify it only processes ImageKit URLs
- Ensure it skips already-transformed URLs
- Test error handling (should not throw)

### Integration Tests:
- Verify warming triggers on video generation completion
- Ensure warming happens on canvas load with existing videos
- Test that warming doesn't interfere with normal video loading

### Manual Testing Steps:
1. Clear browser cache and ImageKit cache (if possible)
2. Generate a new video node
3. Monitor network tab for HEAD requests to transformation URLs
4. Immediately hover over the video node when ready
5. Verify no loading delay (should be instant)
6. Refresh the page
7. Verify HEAD requests are sent again for existing videos
8. Hover over videos and confirm they load instantly

## Performance Considerations

- Pre-warming uses HEAD requests (minimal bandwidth)
- Runs in background via requestIdleCallback (non-blocking)
- Fire-and-forget pattern (doesn't delay other operations)
- Only warms ImageKit URLs (skips other domains)
- Deduplicates URLs before warming (prevents redundant requests)

## Migration Notes

No migration needed. This is a progressive enhancement that:
- Works immediately for new videos
- Improves experience for existing videos on next load
- Falls back gracefully if warming fails

## References

- Original research: `thoughts/shared/research/2025-10-17-ENG-1126-imagekit-video-pregeneration.md`
- ImageKit optimization implementation: `src/lib/videos/video-texture-loader.ts:107-152`
- Existing warming pattern: `src/lib/schema/map-before-request.ts:1748-1760`
- Video node generation hook: `src/hooks/use-node-generation.tsx:106-151`