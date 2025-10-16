# Video Thumbnail Loading Fix Implementation Plan

## Overview

Fix the video thumbnail loading issue where thumbnails only appear on hover instead of loading immediately when the component mounts. The thumbnail loading code is already implemented but videoUrl is being conditionally passed, preventing thumbnails from loading before user interaction.

## Current State Analysis

After reviewing the code, the issue has been identified:

1. **VideoMaterial has proper thumbnail loading** (`src/components/r3f/blocks/video/video-material.tsx:78-126`)
   - Thumbnail loading effect depends only on `[videoUrl]`
   - Should load immediately when videoUrl is available
   - Properly configured with texture settings and disposal

2. **VideoBlockResultR3F is blocking the URL** (`src/components/r3f/video-block-result.tsx:125`)
   - Currently: `videoUrl={shouldLoad ? videoUrl : undefined}`
   - This means VideoMaterial only receives URL after interaction
   - Thumbnails can't load without the URL

### Key Discovery:
The recent change to conditionally pass videoUrl based on `shouldLoad` prevents the thumbnail system from working as designed. The VideoMaterial component needs the URL immediately to load thumbnails, while using `shouldLoadVideo` prop to control video loading.

## Desired End State

Video blocks display thumbnails immediately on render while maintaining lazy loading for videos:
- Thumbnails load as soon as component mounts with videoUrl
- Videos only load after user interaction (hover/selection)
- Smooth transition from thumbnail to video
- No performance regression from lazy loading

### Verification:
- Thumbnails appear immediately without user interaction
- Videos still load lazily on hover/selection
- No increase in initial bandwidth usage (only lightweight thumbnail loads)

## What We're NOT Doing

- NOT modifying the thumbnail loading logic in VideoMaterial (it's already correct)
- NOT changing the lazy loading behavior for videos
- NOT removing the shouldLoadVideo prop
- NOT changing how interaction states are tracked

## Implementation Approach

Revert the conditional URL passing and rely on the `shouldLoadVideo` prop to control video loading while always passing the URL for thumbnail loading.

## Phase 1: Fix URL Passing in VideoBlockResultR3F

### Overview

Restore the original behavior of always passing videoUrl to VideoMaterial, letting the shouldLoadVideo prop control video loading.

### Changes Required:

#### 1. Fix VideoMaterial Props

**File**: `src/components/r3f/video-block-result.tsx`
**Changes**: Restore unconditional videoUrl passing

Current (line 125):
```tsx
videoUrl={shouldLoad ? videoUrl : undefined}
```

Change to:
```tsx
videoUrl={videoUrl}
shouldLoadVideo={shouldLoad}
```

This ensures:
- VideoMaterial always receives the URL (needed for thumbnails)
- Video loading is still controlled by shouldLoadVideo prop
- Thumbnails can load immediately while videos wait for interaction

### Success Criteria:

#### Automated Verification:
- [x] Type checking passes: `pnpm typecheck`
- [x] Linting passes: `pnpm lint`
- [x] Build succeeds: `pnpm build`
- [x] Development server runs without errors: `pnpm dev`

#### Manual Verification:
- [ ] Open a project with video blocks
- [ ] Thumbnails appear immediately without hovering
- [ ] Hover over video block - video starts loading
- [ ] Video plays when fully loaded
- [ ] Check Network tab - thumbnail requests happen immediately
- [ ] Check Network tab - video requests only happen after hover
- [ ] No console errors or warnings

---

## Testing Strategy

### Manual Testing Steps:
1. Create a new project or open existing with video blocks
2. Observe that thumbnails load immediately (no skeleton)
3. Check browser DevTools Network tab:
   - Thumbnail requests (`.jpg` files) should appear immediately
   - Video requests (`.mp4` files) should only appear after hover
4. Hover over video block and verify video loads and plays
5. Test with multiple video blocks on canvas
6. Verify memory usage remains reasonable

### Performance Verification:
- Initial page load should only fetch thumbnails (~50-200KB each)
- Videos should not download until interaction
- No regression in time to interactive

## Implementation Notes

This is a one-line fix that restores the intended behavior. The thumbnail loading system is already fully implemented and just needs the URL to be available.

The root cause was a recent change that tried to optimize by not passing the URL at all when videos shouldn't load, but this prevented the thumbnail system from working.

## References

- Original research: `thoughts/shared/research/2025-10-16-ENG-1057-video-thumbnail-loading-behavior.md`
- Implementation plan: `thoughts/shared/plans/2025-10-16-ENG-1057-video-thumbnails.md`
- Original ticket: `thoughts/shared/tickets/ENG-1057.md`