# Video Lazy Loading Implementation Plan

## Overview

Implement interaction-triggered video loading in the r3f canvas where videos only load when users hover over or select a block, fixing performance issues caused by multiple videos loading simultaneously on page mount.

## Current State Analysis

Videos in the r3f canvas currently load immediately when components mount, causing performance degradation when multiple video blocks are present. While playback correctly waits for hover/selection, the video data itself (entire file) is downloaded immediately via Web Worker or main thread, consuming bandwidth and memory unnecessarily.

### Key Discoveries:

- Play trigger is `selected || isHovered` at `src/components/r3f/video-block-result.tsx:66`
- Video loading starts on mount at `src/components/r3f/blocks/video/video-material.tsx:62-175`
- Web Worker fetches entire video blob at `public/workers/video-processor.worker.js:94`
- Main thread fallback uses `preload="auto"` at `src/lib/videos/video-texture-loader.ts:523`
- No lazy loading patterns currently applied to video blocks

## Desired End State

Videos should only begin loading when a user first hovers over or selects a video block. Once loaded, videos remain in memory for smooth replay. This matches the current play/pause trigger logic while dramatically reducing initial page load.

### Verification:

- Opening a canvas with multiple video blocks shows no video network requests
- Hovering over a video block triggers its download
- Subsequent hovers/selections play immediately without re-downloading
- Performance remains smooth with many video blocks on screen

## What We're NOT Doing

- Not implementing viewport-based loading or unloading
- Not changing the preview thumbnail system
- Not modifying input preview videos in blocks
- Not implementing progressive loading (metadata first)
- Not adding loading progress indicators
- Not changing the Web Worker architecture
- Not implementing automatic memory cleanup/eviction

## Implementation Approach

Add a minimal interaction tracking layer that delays passing the video URL to the loading system until the first hover or selection event. This preserves all existing video infrastructure while adding lazy loading behavior.

## Phase 1: Add Interaction-Based Loading Trigger

### Overview

Add state tracking for first interaction and conditional video URL passing to delay loading until user interaction.

### Changes Required:

#### 1. VideoBlockResultR3F Component

**File**: `src/components/r3f/video-block-result.tsx`
**Changes**: Add interaction tracking state and conditional URL passing

```typescript
// After line 48, add:
const [hasInteracted, setHasInteracted] = useState(false);

// After line 66, add:
const shouldLoad = hasInteracted || selected || isHovered;

// Modify the hover handlers at lines 101-102:
onPointerEnter={() => {
  setIsHovered(true);
  if (!hasInteracted) {
    setHasInteracted(true);
  }
}}
onPointerLeave={() => setIsHovered(false)}

// Add selection effect after line 66:
useEffect(() => {
  if (selected && !hasInteracted) {
    setHasInteracted(true);
  }
}, [selected, hasInteracted]);

// Modify the VideoMaterial prop at line 110-119:
<VideoMaterial
  videoUrl={shouldLoad ? videoUrl : undefined}
  displayWidth={displayWidth}
  displayHeight={displayHeight}
  shouldPlay={shouldPlay}
  blockId={id}
  quality={quality}
  muted={muted}
  onVideoLoaded={handleVideoLoaded}
/>
```

### Success Criteria:

#### Automated Verification:

- [x] Type checking passes: `pnpm typecheck`
- [x] Build succeeds: `pnpm build`
- [x] Unit tests pass: `pnpm test:unit`

#### Manual Verification:

- [ ] Video blocks display without loading videos initially
- [ ] First hover triggers video loading (check Network tab)
- [ ] Selection also triggers loading if not hovered first
- [ ] Subsequent interactions don't trigger duplicate loads

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 2: Handle VideoMaterial Loading States

### Overview

Ensure VideoMaterial properly handles undefined URLs and shows appropriate placeholder state before interaction.

### Changes Required:

#### 1. VideoMaterial Component

**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Handle undefined videoUrl gracefully

```typescript
// Modify the effect at line 62 to handle undefined URL:
useEffect(() => {
  // Early return if no URL provided (not yet interacted)
  if (!videoUrl) {
    setIsLoading(false);
    setTexture(null);
    videoRef.current = null;
    return;
  }

  if (!useWorker) {
    // ... existing fallback logic
    return;
  }

  // ... rest of existing effect code
}, [videoUrl, blockId, quality, useWorker, fallbackLoaded, fallbackTexture, fallbackVideo, fallbackError, onVideoLoaded, displayWidth, displayHeight, muted]);

// Modify the background texture hook call at line 49:
const {
  texture: fallbackTexture,
  video: fallbackVideo,
  isLoaded: fallbackLoaded,
  error: fallbackError,
} = useBackgroundVideoTexture(!useWorker && videoUrl ? videoUrl : undefined, {
  loop: true,
  muted: muted,
  autoplay: false,
});

// Update the render condition at line 209:
if (!videoUrl || isLoading || !texture) {
  return <SkeletonMaterial width={displayWidth} height={displayHeight} />;
}
```

### Success Criteria:

#### Automated Verification:

- [x] Type checking passes: `pnpm typecheck`
- [x] Build succeeds: `pnpm build`

#### Manual Verification:

- [ ] Video blocks show skeleton placeholder before interaction
- [ ] Skeleton smoothly transitions to video after loading
- [ ] No console errors when videoUrl is undefined
- [ ] Loading state properly reflects actual loading status

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 3: Apply to Static Video Blocks

### Overview

Extend the lazy loading pattern to static video blocks for consistency.

### Changes Required:

#### 1. StaticVideoBlockR3F Component

**File**: `src/components/r3f/static-video-block-r3f.tsx`
**Changes**: Add same interaction tracking pattern

```typescript
// After line 72, add:
const [hasInteracted, setHasInteracted] = useState(false);

// After the hover state line, add:
const shouldLoad = hasInteracted || selected || isHovered;

// Add selection effect:
useEffect(() => {
  if (selected && !hasInteracted) {
    setHasInteracted(true);
  }
}, [selected, hasInteracted]);

// Modify hover handlers in the mesh (around line 150):
onPointerEnter={(e) => {
  e.stopPropagation();
  setIsHovered(true);
  if (!hasInteracted) {
    setHasInteracted(true);
  }
}}

// Modify VideoBlockResultR3F props at line 192-206:
<VideoBlockResultR3F
  id={block.id}
  videoUrl={shouldLoad ? videoUrl : undefined}
  selected={selected}
  isDragging={isDragging}
  displayWidth={displayWidth}
  displayHeight={displayHeight}
  quality={videoQuality}
  position={meshPosition}
  muted={muted}
/>
```

### Success Criteria:

#### Automated Verification:

- [x] Type checking passes: `pnpm typecheck`
- [x] Build succeeds: `pnpm build`

#### Manual Verification:

- [ ] Static video blocks also delay loading until interaction
- [ ] Behavior matches dynamic video blocks
- [ ] No regressions in static block functionality

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 4: Optimize VideoTextureCoordinator

### Overview

Add support for lazy initialization in the coordinator to prevent worker preloading when no videos will load immediately.

### Changes Required:

#### 1. VideoTextureCoordinator Class

**File**: `src/lib/videos/video-texture-loader.ts`
**Changes**: Make worker preloading conditional

```typescript
// Modify the constructor at line 98-99:
constructor() {
  // Don't automatically preload worker
  // void this.preloadWorker();  // Comment out or remove
}

// Add lazy initialization check in loadVideoTexture at line 344:
async loadVideoTexture(
  videoUrl: string,
  blockId: string,
  options: VideoLoadOptions = {},
): Promise<VideoTextureResult> {
  // Initialize on first actual load request
  if (!this.initPromise && !this.isInitialized) {
    this.initPromise = this.initialize();
  }

  // ... rest of existing method
}
```

### Success Criteria:

#### Automated Verification:

- [x] Type checking passes: `pnpm typecheck`
- [x] Build succeeds: `pnpm build`

#### Manual Verification:

- [ ] Web Worker doesn't preload on page load
- [ ] Worker initializes on first video interaction
- [ ] Subsequent videos load normally after initialization
- [ ] Fallback to main thread still works if worker fails

---

## Testing Strategy

### Unit Tests:

- Test VideoBlockResultR3F with/without interaction
- Test VideoMaterial with undefined URL
- Mock hover and selection events
- Verify loading triggers only once per video

### Integration Tests:

- Create canvas with multiple video blocks
- Verify no video requests on initial load
- Test hover triggers single video load
- Test selection triggers load if not hovered
- Verify cached videos don't reload

### Manual Testing Steps:

1. Open a canvas with 5+ video blocks
2. Open browser Network tab, filter by media
3. Verify no video requests on page load
4. Hover over first video block
5. Verify single video request starts
6. Wait for video to load and play
7. Leave hover, then hover again
8. Verify no duplicate request
9. Select a different video block
10. Verify that video loads
11. Test with slow network (throttled connection)
12. Verify all videos eventually load on interaction

## Performance Considerations

### Expected Improvements:

- Initial page load: 80-90% reduction in bandwidth usage with multiple videos
- Time to interactive: Significantly faster with no concurrent video loads
- Memory usage: Gradual increase as videos are interacted with
- Worker initialization: Delayed until first video interaction

### Tradeoffs:

- First interaction latency: Videos start loading on hover (acceptable UX)
- Memory growth: Videos remain loaded after interaction (prevents re-downloading)
- No preloading: Network idle time not utilized for prefetching

## Migration Notes

No data migration required. The changes are backward compatible and only affect runtime loading behavior. Existing video blocks will continue to work with the new lazy loading automatically applied.

## References

- Original ticket: ENG-1057 "Video slow loading, video load failed on many canvases"
- Research document: `thoughts/shared/research/2025-01-14-ENG-1057-video-loading-r3f-canvas.md`
- Current video play logic: `src/components/r3f/video-block-result.tsx:66`
- Video loading implementation: `src/components/r3f/blocks/video/video-material.tsx:62-175`
- Web Worker video fetching: `public/workers/video-processor.worker.js:94`
