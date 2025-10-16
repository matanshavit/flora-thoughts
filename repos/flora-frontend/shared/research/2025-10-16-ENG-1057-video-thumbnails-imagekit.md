---
date: 2025-10-16T13:07:06-04:00
researcher: Claude Code
git_commit: 8eae7a89b9d8310be18bfbde842460471c84d543
branch: eng-1057-video
repository: eng-1057-video
topic: "ENG-1057: Video thumbnails for blank tiles using ImageKit"
tags: [research, codebase, video-node, image-node, imagekit, thumbnails]
status: complete
last_updated: 2025-10-16
last_updated_by: Claude Code
---

# Research: ENG-1057 - Video Thumbnails for Blank Tiles Using ImageKit

**Date**: 2025-10-16T13:07:06-04:00
**Researcher**: Claude Code
**Git Commit**: 8eae7a89b9d8310be18bfbde842460471c84d543
**Branch**: eng-1057-video
**Repository**: eng-1057-video

## Research Question

ENG-1057: Now that videos do not load on page load, those tiles are blank until they are hovered and the videos load. Look at the video node, the image node which has an image on it, and read https://imagekit.io/docs/create-video-thumbnails including the basic example to see how to do this with ImageKit.

## Summary

The current video node implementation displays a pulsating skeleton placeholder until videos are hovered, at which point they load and play. While the `VideoUrlOutput` type already includes a `thumbnailUrl` field, it is not currently utilized. The image node successfully displays static thumbnails using ImageKit's URL transformation, and ImageKit provides a simple `/ik-thumbnail.jpg` suffix pattern to generate video thumbnails. To resolve the blank tile issue, video thumbnails can be generated using ImageKit's transformation and displayed similarly to how image nodes show previews.

## Detailed Findings

### Current Video Loading Behavior

**Video Block Implementation** (`src/components/r3f/video-block-result.tsx:31-128`)
- Videos load immediately on component mount, not on hover
- Hover state only controls playback (play/pause), not loading
- Shows `SkeletonMaterial` with pulsating gray animation while loading
- Video URLs stored in `node.data.output.videoUrl`

**Skeleton Placeholder** (`src/components/r3f/blocks/shaders/skeleton.tsx:52-90`)
- Displays pulsating gray rectangle during video load
- Animation: `float pulse = 0.4 + 0.2 * sin(time * 3.0)`
- Background color: `(0.55, 0.55, 0.55)`
- Applied when `isLoading || !texture` in VideoMaterial

**Existing Thumbnail Support** (`src/components/nodes/types.ts:88-92`)
```typescript
export type VideoUrlOutput = BaseOutput & {
  videoUrl?: string;
  thumbnailUrl?: string;  // Already exists but unused
  aspectRatio?: number | null;
};
```

### Image Node Thumbnail Display Pattern

**Image Result Component** (`src/components/nodes/models/image-block/image-block-result.tsx`)
- Uses Next.js `Image` component with `toHumaneSizedImageUrl()` for optimization
- Shows `Skeleton` overlay while loading
- Smooth opacity transition when loaded (opacity-0 → opacity-100)
- Calculates aspect ratio from naturalWidth/naturalHeight

**ThumbnailImage Component** (`src/components/ui/thumbnail-image.tsx`)
- Reusable thumbnail component with retry logic (20 retries, 1000ms delay)
- Default size: 48x48px
- Uses `toHumaneSizedImageUrl()` for ImageKit transformations
- For non-image URLs: appends `/ik-thumbnail.jpg`
- Shows loading placeholder with pulse animation

**URL Transformation Utility** (`src/lib/utils.ts:233-242`)
```typescript
function toHumaneSizedImageUrl(url, { width, height }) {
  if (url.startsWith("https://ik.imagekit.io/")) {
    return `${url}?tr=w-${width},h-${height}`;
  }
  return url;
}
```

### ImageKit Video Thumbnail Capabilities

**Basic Implementation**
- Append `/ik-thumbnail.jpg` to video URL for first frame
- Example: `https://ik.imagekit.io/demo/sample-video.mp4/ik-thumbnail.jpg`

**Frame Selection**
- Use `so` (start offset) parameter for specific timestamp
- Example: `video.mp4/ik-thumbnail.jpg?tr=so-5` (5-second mark)

**Image Transformations**
- Width: `tr=w-300`
- Height: `tr=h-200`
- Aspect ratio: `tr=ar-16-9`
- Combined: `video.mp4/ik-thumbnail.jpg?tr=w-300,h-200`

### Key Video Node Files

**2D Canvas Rendering** (`src/components/nodes/models/video-block/result-state.tsx`)
- HTML `<video>` element with `preload="metadata"`
- Shows skeleton during load
- Auto-plays on hover/selection
- Mute/unmute button overlay

**3D Canvas Rendering** (`src/components/r3f/video-block-result.tsx`)
- Three.js VideoTexture via VideoMaterial
- Suspense boundary with skeleton fallback
- Hover detection via mesh pointer events
- Frame invalidation during playback

**Video Material Loader** (`src/components/r3f/blocks/video/video-material.tsx`)
- Web Worker loading with main thread fallback
- Retry logic: 3 attempts, 1000ms delay
- Creates THREE.VideoTexture on canplay event
- Manages play/pause based on `shouldPlay` prop

## Code References

- `src/components/nodes/types.ts:89` - `thumbnailUrl` field already exists in VideoUrlOutput
- `src/components/r3f/video-block-result.tsx:108-121` - Current skeleton fallback during video load
- `src/components/ui/thumbnail-image.tsx:48-60` - Existing logic for appending `/ik-thumbnail.jpg` to non-image URLs
- `src/lib/utils.ts:233-242` - `toHumaneSizedImageUrl()` function for ImageKit transformations
- `src/components/nodes/models/image-block/image-block-result.tsx:50-65` - Image node pattern for displaying thumbnails

## Architecture Documentation

### Current Video Display Flow
1. Component mounts with `videoUrl` prop
2. Shows SkeletonMaterial placeholder immediately
3. VideoMaterial loads video texture via Web Worker
4. On texture ready, replaces skeleton with video
5. Video remains paused until hover/selection
6. Hover triggers play(), leaving hover triggers pause()

### Image Node Display Pattern (Reference)
1. Component receives `imageUrl` prop
2. Shows Skeleton overlay while loading
3. Loads image with opacity-0
4. On load: calculates aspect ratio, sets opacity-100
5. Smooth transition with `transition-opacity duration-200`

### ImageKit URL Pattern
1. Video URL: `https://ik.imagekit.io/{path}/video.mp4`
2. Thumbnail: `https://ik.imagekit.io/{path}/video.mp4/ik-thumbnail.jpg`
3. With sizing: `https://ik.imagekit.io/{path}/video.mp4/ik-thumbnail.jpg?tr=w-352,h-352`
4. With timestamp: `https://ik.imagekit.io/{path}/video.mp4/ik-thumbnail.jpg?tr=so-2`

## Implementation Approach

Based on the research, the solution involves:

1. **Generate Thumbnail URLs**
   - When `videoUrl` exists, create thumbnail URL by appending `/ik-thumbnail.jpg`
   - Apply appropriate transformations (width, height based on node size)
   - Optionally use `so` parameter for specific frame (e.g., 2 seconds in)

2. **Display Thumbnail Instead of Skeleton**
   - Replace SkeletonMaterial with image display while video loads
   - Use similar pattern to image nodes: show thumbnail with loading state
   - Keep thumbnail visible until video texture is ready

3. **Leverage Existing Infrastructure**
   - Use `ThumbnailImage` component which already handles `/ik-thumbnail.jpg` logic
   - Apply `toHumaneSizedImageUrl()` for optimal sizing
   - Store generated thumbnail URL in existing `thumbnailUrl` field

4. **Maintain Current Behavior**
   - Keep video loading on mount (not lazy)
   - Preserve hover-to-play functionality
   - Show thumbnail during load instead of skeleton

## Open Questions

1. Should thumbnail generation happen server-side (during video generation) or client-side (on display)?
2. What timestamp should be used for thumbnails (first frame vs 2-3 seconds in)?
3. Should thumbnails be cached/stored in the database to avoid repeated generation?
4. How to handle videos that aren't from ImageKit (fallback behavior)?