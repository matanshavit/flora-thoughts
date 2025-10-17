# ImageKit Video Pregeneration for R3F Canvas Implementation Plan

## Overview

Implement a reusable hook that pre-warms ImageKit video transformations during browser idle time to eliminate the initial 2-3 second loading delay when users first interact with videos in the R3F canvas.

## Current State Analysis

The R3F canvas video system currently applies ImageKit transformations (WebM format, quality presets) only when videos are loaded on user interaction. The first request to a transformation URL triggers ImageKit to generate the transformed video, causing a delay. Subsequent requests are served instantly from ImageKit's CDN cache.

### Key Discoveries:
- ImageKit transformations are applied via URL parameters at `src/lib/videos/video-texture-loader.ts:107-152`
- Videos only load when `shouldLoadVideo` becomes true (hover/select) at `src/components/r3f/blocks/video/video-material.tsx:69`
- Existing warming pattern exists at `src/lib/schema/map-before-request.ts:1748-1760`
- R3F preloader uses requestIdleCallback pattern at `src/lib/r3f/preloader.tsx:176-191`
- No current mechanism gathers or pre-warms video URLs during scene initialization

## Desired End State

After implementation, when an R3F canvas with videos loads:
1. Video URLs are identified and transformed with medium quality preset
2. During browser idle time, HEAD requests are sent to warm ImageKit transformations
3. When users interact with videos, they load instantly without transformation delay
4. The warming process is non-blocking and doesn't affect initial page load performance

### Verification:
- Network tab shows HEAD requests to transformed video URLs during idle time
- First user interaction with videos loads without the 2-3 second delay
- ImageKit dashboard shows transformations are pre-generated and cached

## What We're NOT Doing

- NOT warming all quality presets (only medium quality)
- NOT blocking on transformation completion (fire-and-forget)
- NOT modifying the ReactFlow canvas implementation
- NOT changing the existing video loading flow or worker logic
- NOT implementing upload-time post-transformations
- NOT adding polling or retry logic for warming

## Implementation Approach

Create a minimal, reusable React hook that leverages requestIdleCallback to warm video transformations in the background. The hook will be integrated into the VideoMaterial component with minimal changes, ensuring transformed videos are cached by ImageKit before user interaction.

## Phase 1: Create Video Warming Hook

### Overview

Create a new React hook that accepts video URLs and warms their ImageKit transformations during browser idle time.

### Changes Required:

#### 1. Create the useVideoWarming Hook

**File**: `src/components/r3f/hooks/use-video-warming.ts` (new file)
**Changes**: Create new hook for video URL warming

```typescript
import { useEffect, useRef } from "react";

interface UseVideoWarmingOptions {
  quality?: "low" | "medium" | "high";
  enabled?: boolean;
}

/**
 * Hook to pre-warm ImageKit video transformations during browser idle time.
 * Sends fire-and-forget HEAD requests to trigger ImageKit transformation caching.
 */
export function useVideoWarming(
  videoUrl: string | undefined,
  options: UseVideoWarmingOptions = {}
) {
  const { quality = "medium", enabled = true } = options;
  const warmedUrlsRef = useRef<Set<string>>(new Set());

  useEffect(() => {
    if (!enabled || !videoUrl) return;

    // Only process ImageKit URLs
    if (!videoUrl.startsWith("https://ik.imagekit.io/")) return;

    // Skip if URL already has transformations
    if (videoUrl.includes("?tr=") || videoUrl.includes("/tr:")) return;

    // Apply quality transformation
    const qualityTransforms: Record<string, string> = {
      low: "f-webm,h-480,c-at_max,q-40,du-10",
      medium: "f-webm,h-720,c-at_max,q-50,du-10",
      high: "f-webm,h-1080,c-at_max,q-60,du-10",
    };

    const transform = qualityTransforms[quality];
    const separator = videoUrl.includes("?") ? "&" : "?";
    const transformedUrl = `${videoUrl}${separator}tr=${transform}`;

    // Skip if already warmed
    if (warmedUrlsRef.current.has(transformedUrl)) return;

    // Function to send warming request
    const warmUrl = () => {
      // Fire-and-forget HEAD request
      fetch(transformedUrl, {
        method: "HEAD",
        cache: "no-store", // Ensure we hit ImageKit, not browser cache
      }).catch(() => {
        // Ignore errors - this is best-effort warming
      });

      // Mark as warmed
      warmedUrlsRef.current.add(transformedUrl);

      // Clean up old entries if set gets too large
      if (warmedUrlsRef.current.size > 100) {
        const entries = Array.from(warmedUrlsRef.current);
        warmedUrlsRef.current = new Set(entries.slice(-50));
      }
    };

    // Use requestIdleCallback for non-blocking warming
    if ("requestIdleCallback" in window) {
      const idleCallbackId = requestIdleCallback(warmUrl, {
        timeout: 5000, // Maximum wait time of 5 seconds
      });

      return () => {
        if ("cancelIdleCallback" in window) {
          cancelIdleCallback(idleCallbackId);
        }
      };
    } else {
      // Fallback for Safari/older browsers
      const timeoutId = setTimeout(warmUrl, 1000);
      return () => clearTimeout(timeoutId);
    }
  }, [videoUrl, quality, enabled]);
}
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compilation passes: `pnpm typecheck`
- [ ] Linting passes: `pnpm lint`
- [ ] Unit tests pass: `pnpm test:unit`

#### Manual Verification:
- [ ] Hook exports correctly and can be imported
- [ ] No TypeScript errors in the new file
- [ ] File follows project conventions

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 2: Add Warming Method to VideoTextureCoordinator

### Overview

Add a public warming method to the VideoTextureCoordinator that can be called by the hook, reusing existing URL transformation logic.

### Changes Required:

#### 1. Add Warming Method to VideoTextureCoordinator

**File**: `src/lib/videos/video-texture-loader.ts`
**Changes**: Add new method for warming video URLs

```typescript
  /**
   * Pre-warm an ImageKit video transformation by sending a HEAD request.
   * This triggers ImageKit to generate and cache the transformation.
   * @param videoUrl - The video URL to warm
   * @param quality - Quality preset to warm (default: "medium")
   * @returns Promise that resolves when request is sent (fire-and-forget)
   */
  async warmVideoUrl(
    videoUrl: string,
    quality: "low" | "medium" | "high" = "medium"
  ): Promise<void> {
    // Apply the same optimization as in loadVideoTexture
    const optimizedUrl = this.getOptimizedVideoUrl(videoUrl, quality);

    // If URL wasn't optimized (not ImageKit or already transformed), skip
    if (optimizedUrl === videoUrl) {
      return;
    }

    try {
      // Fire-and-forget HEAD request to trigger transformation
      await fetch(optimizedUrl, {
        method: "HEAD",
        cache: "no-store", // Ensure we hit ImageKit, not browser cache
        signal: AbortSignal.timeout(5000), // 5 second timeout
      });
    } catch {
      // Ignore errors - this is best-effort warming
      // Errors could be network issues, CORS, or timeouts
      // The actual video load will handle errors properly
    }
  }
```

#### 2. Update Hook to Use Coordinator Method

**File**: `src/components/r3f/hooks/use-video-warming.ts`
**Changes**: Update to use the coordinator's warming method

```typescript
import { useEffect, useRef } from "react";
import { videoCoordinator } from "@/lib/videos/video-texture-loader";

interface UseVideoWarmingOptions {
  quality?: "low" | "medium" | "high";
  enabled?: boolean;
}

/**
 * Hook to pre-warm ImageKit video transformations during browser idle time.
 * Uses the VideoTextureCoordinator to ensure consistent URL transformation.
 */
export function useVideoWarming(
  videoUrl: string | undefined,
  options: UseVideoWarmingOptions = {}
) {
  const { quality = "medium", enabled = true } = options;
  const warmedUrlsRef = useRef<Set<string>>(new Set());

  useEffect(() => {
    if (!enabled || !videoUrl) return;

    // Skip if already warmed this URL with this quality
    const warmingKey = `${videoUrl}:${quality}`;
    if (warmedUrlsRef.current.has(warmingKey)) return;

    // Function to send warming request
    const warmUrl = () => {
      // Use coordinator for consistent URL transformation
      videoCoordinator.warmVideoUrl(videoUrl, quality);

      // Mark as warmed
      warmedUrlsRef.current.add(warmingKey);

      // Clean up old entries if set gets too large
      if (warmedUrlsRef.current.size > 100) {
        const entries = Array.from(warmedUrlsRef.current);
        warmedUrlsRef.current = new Set(entries.slice(-50));
      }
    };

    // Use requestIdleCallback for non-blocking warming
    if ("requestIdleCallback" in window) {
      const idleCallbackId = requestIdleCallback(warmUrl, {
        timeout: 5000, // Maximum wait time of 5 seconds
      });

      return () => {
        if ("cancelIdleCallback" in window) {
          cancelIdleCallback(idleCallbackId);
        }
      };
    } else {
      // Fallback for Safari/older browsers
      const timeoutId = setTimeout(warmUrl, 1000);
      return () => clearTimeout(timeoutId);
    }
  }, [videoUrl, quality, enabled]);
}
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compilation passes: `pnpm typecheck`
- [ ] Linting passes: `pnpm lint`
- [ ] Build succeeds: `pnpm build`
- [ ] Unit tests pass: `pnpm test:unit`

#### Manual Verification:
- [ ] VideoTextureCoordinator exports the new warmVideoUrl method
- [ ] Method reuses existing URL transformation logic correctly
- [ ] No regression in existing video loading functionality

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 3: Integrate Hook into VideoMaterial Component

### Overview

Minimally integrate the useVideoWarming hook into the VideoMaterial component to warm videos when they mount.

### Changes Required:

#### 1. Add Hook to VideoMaterial Component

**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Import and use the warming hook

Add import at the top of the file (after other imports):
```typescript
import { useVideoWarming } from "@/components/r3f/hooks/use-video-warming";
```

Add hook usage inside the VideoMaterial component (after existing hooks, around line 67):
```typescript
  // Pre-warm the video transformation during idle time
  useVideoWarming(videoUrl, {
    quality: quality || "medium",
    enabled: true,
  });
```

The complete context around line 67 should look like:
```typescript
export const VideoMaterial = ({
  videoUrl,
  thumbnailUrl,
  displayWidth,
  displayHeight,
  shouldPlay,
  shouldLoadVideo,
  blockId,
  quality = "medium",
  useWorker = true,
  muted = false,
  onVideoLoaded,
  onVideoDurationChange,
}: VideoMaterialProps) => {
  const [texture, setTexture] = useState<THREE.VideoTexture | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const videoElementRef = useRef<HTMLVideoElement | null>(null);
  const videoRef = useCallbackRef<HTMLVideoElement>((video) => {
    videoElementRef.current = video;
  });

  // Existing hooks...
  const { texture: fallbackTexture, isLoaded: fallbackLoaded } =
    useBackgroundVideoTexture(
      !useWorker && shouldLoadVideo && !texture ? videoUrl : undefined,
      {
        quality,
        onVideoLoaded: (video) => {
          onVideoLoaded?.(video);
        },
      }
    );

  // Pre-warm the video transformation during idle time
  useVideoWarming(videoUrl, {
    quality: quality || "medium",
    enabled: true,
  });

  // Rest of component...
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compilation passes: `pnpm typecheck`
- [ ] Linting passes: `pnpm lint`
- [ ] Build succeeds: `pnpm build`
- [ ] Development server runs without errors: `pnpm dev`

#### Manual Verification:
- [ ] Open browser DevTools Network tab
- [ ] Load an R3F canvas with video blocks
- [ ] Observe HEAD requests to transformed video URLs (with `?tr=` parameters) during idle time
- [ ] HEAD requests complete with 200 or 302 status (both are fine)
- [ ] Hover over a video block - it should load faster than before
- [ ] No console errors or warnings related to warming
- [ ] Existing video functionality works as before

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 4: Testing and Performance Verification

### Overview

Verify that the implementation successfully pre-warms video transformations and improves load times without impacting performance.

### Changes Required:

No code changes in this phase - purely testing and verification.

### Success Criteria:

#### Automated Verification:
- [ ] All unit tests pass: `pnpm test:unit`
- [ ] Type checking passes: `pnpm typecheck`
- [ ] Linting passes: `pnpm lint`
- [ ] Build completes successfully: `pnpm build`

#### Manual Verification:

1. **Warming Verification**:
   - [ ] Open DevTools Network tab and filter by "HEAD" requests
   - [ ] Load a project with multiple video blocks in R3F canvas
   - [ ] Confirm HEAD requests are sent to ImageKit URLs with transformations
   - [ ] Requests should have `?tr=f-webm,h-720,c-at_max,q-50,du-10` parameters
   - [ ] Requests happen during idle time (not immediately on page load)

2. **Performance Testing**:
   - [ ] Initial page load is not slowed down by warming
   - [ ] HEAD requests don't block user interactions
   - [ ] Browser remains responsive during warming

3. **Load Time Improvement**:
   - [ ] First test (cold cache):
     - Clear browser cache and cookies
     - Load project with videos
     - Wait for idle warming to complete (check Network tab)
     - Hover over video - note load time
   - [ ] Second test (warm cache):
     - Refresh the page
     - Immediately hover over video
     - Video should load noticeably faster (no 2-3 second delay)

4. **Edge Cases**:
   - [ ] Non-ImageKit videos still work (no warming, normal loading)
   - [ ] Videos with existing transformations aren't re-warmed
   - [ ] Component unmounting doesn't cause errors
   - [ ] Multiple videos warm correctly without duplicates

5. **Browser Compatibility**:
   - [ ] Test in Chrome (should use requestIdleCallback)
   - [ ] Test in Safari (should use setTimeout fallback)
   - [ ] Test in Firefox (should use requestIdleCallback)

## Testing Strategy

### Unit Tests:

Create a test file for the new hook:

**File**: `src/components/r3f/hooks/use-video-warming.test.ts` (new file)

```typescript
import { renderHook } from "@testing-library/react";
import { useVideoWarming } from "./use-video-warming";
import { vi } from "vitest";

describe("useVideoWarming", () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it("should not warm non-ImageKit URLs", () => {
    const fetchSpy = vi.spyOn(global, "fetch");
    renderHook(() =>
      useVideoWarming("https://example.com/video.mp4")
    );
    expect(fetchSpy).not.toHaveBeenCalled();
  });

  it("should not warm URLs with existing transformations", () => {
    const fetchSpy = vi.spyOn(global, "fetch");
    renderHook(() =>
      useVideoWarming("https://ik.imagekit.io/test/video.mp4?tr=f-webm")
    );
    expect(fetchSpy).not.toHaveBeenCalled();
  });

  // Add more tests as needed
});
```

### Manual Testing Steps:

1. Start development server: `pnpm dev`
2. Open project with video blocks
3. Open DevTools (Network tab, filter by "HEAD")
4. Observe warming requests during idle time
5. Interact with videos and verify improved load times
6. Test in multiple browsers

## Performance Considerations

- HEAD requests are lightweight (no body download)
- requestIdleCallback ensures warming doesn't block user interactions
- Fire-and-forget pattern prevents blocking on network issues
- Maximum 100 URLs tracked to prevent memory leaks
- 5-second timeout prevents hanging requests

## Migration Notes

This is a progressive enhancement - no migration needed. Existing videos will continue to work as before, with the added benefit of pre-warming for faster loads.

## References

- Original research: `thoughts/shared/research/2025-10-17-ENG-1126-imagekit-video-pregeneration.md`
- Current video loading: `src/components/r3f/blocks/video/video-material.tsx:69-182`
- VideoTextureCoordinator: `src/lib/videos/video-texture-loader.ts:83-708`
- Existing warming pattern: `src/lib/schema/map-before-request.ts:1748-1760`
- R3F preloader pattern: `src/lib/r3f/preloader.tsx:176-191`