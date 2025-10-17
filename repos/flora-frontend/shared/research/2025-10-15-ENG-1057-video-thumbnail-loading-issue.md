---
date: 2025-10-15T23:50:18-04:00
researcher: matanshavit
git_commit: 85f4893d3984e569b46f24d9240bba40def7d96c
branch: eng-1057-video
repository: flora-frontend
topic: "ENG-1057 PR 2075 Video Thumbnail Loading Issue"
tags: [research, codebase, video-loading, lazy-loading, r3f-canvas, thumbnails]
status: complete
last_updated: 2025-10-15
last_updated_by: matanshavit
---

# Research: ENG-1057 PR 2075 Video Thumbnail Loading Issue

**Date**: 2025-10-15T23:50:18-04:00
**Researcher**: matanshavit
**Git Commit**: 85f4893d3984e569b46f24d9240bba40def7d96c
**Branch**: eng-1057-video
**Repository**: flora-frontend

## Research Question

ENG-1057: On PR 2075 we need to change the initial block to load the image thumbnail, but not the video. In this PR, it loads neither on page load.

## Summary

PR 2075 successfully implements lazy loading for video blocks in the R3F canvas, preventing videos from loading until user interaction (hover/selection). However, the current implementation shows a skeleton placeholder material instead of the video's thumbnail image on initial load. The requirement is to display the video thumbnail immediately while deferring the actual video loading until interaction.

## Detailed Findings

### Current Implementation in PR 2075

#### Lazy Loading Mechanism (`src/components/r3f/static-video-block-r3f.tsx`)

The implementation uses a dual-component approach with interaction tracking:

- **Line 79**: `const [hasInteracted, setHasInteracted] = useState(false)` - Tracks if user has ever interacted with the block
- **Line 110**: `const shouldLoad = hasInteracted || selected || isHovered` - Determines when to load video
- **Line 214**: Passes `videoUrl={shouldLoad ? videoUrl : undefined}` to child component
- **Lines 113-124**: Effects that set `hasInteracted` to true when block is selected or hovered

When `shouldLoad` is false, the component passes `undefined` instead of the video URL, preventing any video loading.

#### Video Block Result Component (`src/components/r3f/video-block-result.tsx`)

Duplicates the interaction tracking logic:

- **Line 49**: Maintains its own `hasInteracted` state
- **Line 68**: `const shouldLoad = hasInteracted || selected || isHovered`
- **Line 125**: Passes `videoUrl={shouldLoad ? videoUrl : undefined}` to VideoMaterial
- **Lines 110-115**: onPointerEnter handler sets interaction state

This provides double-gating to ensure videos don't load prematurely.

#### Video Material Behavior (`src/components/r3f/blocks/video/video-material.tsx`)

The critical behavior when no URL is provided:

- **Lines 63-69**: Early return when `!videoUrl`:
  ```typescript
  if (!videoUrl) {
    setIsLoading(false);
    setTexture(null);
    videoRef.current = null;
    return;
  }
  ```
- **Lines 211-220**: Returns SkeletonMaterial when no URL or loading:
  ```typescript
  if (!videoUrl || isLoading || !texture) {
    return (
      <SkeletonMaterial
        width={displayWidth}
        height={displayHeight}
        radiusX={BLOCK_BORDER_RADIUS}
        radiusY={BLOCK_BORDER_RADIUS}
      />
    );
  }
  ```

### The Issue: Missing Thumbnail Display

The current implementation shows `SkeletonMaterial` (an animated pulsing placeholder) instead of the actual video thumbnail when `videoUrl` is undefined. There's no mechanism to:

1. Generate or retrieve a thumbnail URL from the video URL
2. Load and display that thumbnail as a texture before video loading
3. Transition from thumbnail to video when interaction occurs

### Existing Thumbnail Infrastructure

The codebase has infrastructure for video thumbnails that isn't being utilized:

#### Thumbnail URL Generation (`shared/files/thumbnails.ts`)

- **Lines 7-23**: Functions to generate ImageKit thumbnail URLs:
  ```typescript
  export function getOutputThumnailUrl(rawOutputUrl: string, size: number = THUMBNAIL_SIZE.SMALL) {
    if (!rawOutputUrl.startsWith("https://ik.imagekit.io/")) {
      return rawOutputUrl;
    }
    return `${rawOutputUrl.endsWith("mp4") ? rawOutputUrl + "/ik-thumbnail.jpg" : rawOutputUrl}?tr=w-${size},h-${size}`;
  }
  ```
- Appends `/ik-thumbnail.jpg` to video URLs for ImageKit CDN
- Supports size transformations via query parameters

#### Video Frame Extraction (`src/lib/videos/helpers.ts`)

- **Lines 98-188**: `getImageCollageFromVideoUrls` extracts frames using ImageKit transformations:
  - Uses `so-<seconds>` transformation to seek to specific timestamps
  - Example: `${baseUrl}/tr:so-1.50,w-480/${filename}/ik-thumbnail.jpg`

#### Thumbnail Display Pattern (`src/components/r3f/shared-asset-preview.tsx`)

- **Lines 149-153**: Shows video frame at 10% duration as thumbnail:
  ```typescript
  video.addEventListener("loadedmetadata", () => {
    video.currentTime = Math.min(1, video.duration * ASSET_VIDEO_PREVIEW_TIME_RATIO);
    handleSetUVTransform(video.videoWidth, video.videoHeight);
    setTexture(videoTexture);
  });
  ```

### What's Needed

The VideoMaterial component needs to:

1. **Accept a thumbnail URL** in addition to video URL
2. **Load thumbnail texture** immediately when component mounts
3. **Display thumbnail** while `!shouldLoad` (before interaction)
4. **Transition to video** when `shouldLoad` becomes true
5. **Show skeleton only** if thumbnail fails to load

The parent components need to:

1. **Generate thumbnail URL** from video URL using existing helpers
2. **Pass both URLs** to VideoMaterial (thumbnail always, video conditionally)
3. **Maintain current lazy loading logic** for video

## Code References

### Current Implementation Files

- `src/components/r3f/static-video-block-r3f.tsx:79-214` - Parent component with interaction tracking
- `src/components/r3f/video-block-result.tsx:49-125` - Child component with duplicate tracking
- `src/components/r3f/blocks/video/video-material.tsx:63-220` - Material showing skeleton instead of thumbnail
- `src/lib/videos/video-texture-loader.ts:335-587` - Video loading coordinator

### Thumbnail Infrastructure (Not Currently Used)

- `shared/files/thumbnails.ts:7-23` - Thumbnail URL generation functions
- `src/lib/videos/helpers.ts:98-188` - Video frame extraction utilities
- `src/components/r3f/shared-asset-preview.tsx:149-153` - Example of thumbnail display pattern

## Architecture Documentation

### Current Loading Flow

1. **Initial State**: VideoBlock renders with `hasInteracted=false`
2. **No URL Passed**: `videoUrl={undefined}` passed to VideoMaterial
3. **Skeleton Display**: VideoMaterial shows SkeletonMaterial placeholder
4. **User Interaction**: Hover/select triggers `hasInteracted=true`
5. **URL Passed**: `videoUrl={actualUrl}` passed to VideoMaterial
6. **Video Loads**: VideoTextureCoordinator loads video via worker
7. **Video Displays**: Shader material renders video texture

### Required Loading Flow

1. **Initial State**: VideoBlock renders with `hasInteracted=false`
2. **Thumbnail URL Generated**: Create thumbnail URL from video URL
3. **Thumbnail Display**: Load and show thumbnail texture immediately
4. **User Interaction**: Hover/select triggers `hasInteracted=true`
5. **Video Loads**: Load video while keeping thumbnail visible
6. **Smooth Transition**: Replace thumbnail with video when ready

### Loading State Matrix

| hasInteracted | videoUrl passed | Current Display  | Required Display  |
| ------------- | --------------- | ---------------- | ----------------- |
| false         | undefined       | SkeletonMaterial | Thumbnail Image   |
| false→true    | undefined→url   | Skeleton→Loading | Thumbnail→Loading |
| true          | url             | Video            | Video             |

## Historical Context (from thoughts/)

Previous research documents identified:

- `thoughts/shared/research/2025-01-14-ENG-1057-video-loading-r3f-canvas.md` - Analysis of video loading architecture
- `thoughts/shared/plans/2025-01-14-ENG-1057-video-lazy-loading.md` - Implementation plan for lazy loading
- `thoughts/shared/research/2025-01-13-ENG-1057-video-webworker-timeout-behavior.md` - WebWorker timeout issues

These documents focus on the lazy loading implementation but don't address the thumbnail display requirement.

## Related Research

- `thoughts/shared/tickets/ENG-1057.md` - Original ticket documentation
- `thoughts/shared/prs/2075_description.md` - PR description for lazy loading implementation

## Open Questions

1. Should thumbnails be generated on-demand or pre-generated and stored?
2. What resolution should thumbnails use for optimal performance vs quality?
3. Should thumbnail loading also use the LazyMediaLoader system or load immediately?
4. How to handle videos not hosted on ImageKit (no automatic thumbnail generation)?
5. Should we cache thumbnail textures separately from video textures?
