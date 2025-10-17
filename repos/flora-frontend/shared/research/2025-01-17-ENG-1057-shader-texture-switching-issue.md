---
date: "2025-10-17T00:16:45-04:00"
researcher: matanshavit
git_commit: d21bc5174bfad6f7b8e3452ec121692d27241834
branch: eng-1057
repository: flora-frontend
topic: "Shader texture switching issue - image texture not replaced by video texture"
tags: [research, codebase, video-material, texture-switching, three-js, react-three-fiber]
status: complete
last_updated: 2025-01-17
last_updated_by: matanshavit
---

# Research: Shader Texture Switching Issue - Image Texture Not Replaced by Video Texture

**Date**: 2025-10-17T00:16:45-04:00
**Researcher**: matanshavit
**Git Commit**: d21bc5174bfad6f7b8e3452ec121692d27241834
**Branch**: eng-1057
**Repository**: flora-frontend

## Research Question
The last commit implements a video lazy loading solution where a thumbnail texture should be replaced by a video texture when the video loads. However, the shader does not switch from the image texture to the video texture. It can render the image texture alone, and when the image texture is completely removed, it can render the video texture alone. But in the current implementation, it does not remove the image texture to show the video texture. Why?

## Summary

The issue stems from the initialization and update logic of the `displayTexture` state in `video-material.tsx`. The `displayTexture` state is initialized with the current value of `thumbnailTexture` (which is `null` at mount time) and the effect that updates it has a logical gap: when both `texture` and `thumbnailTexture` are `null`, the effect runs but takes no action, leaving `displayTexture` unchanged. Additionally, there's a race condition where state updates from async callbacks are batched, but effect execution is deferred. This causes the component to render with `isLoading = false` and `texture` set, but `displayTexture` not yet updated, resulting in the shader continuing to use the old texture value.

## Detailed Findings

### The displayTexture State Initialization Problem

At `src/components/r3f/blocks/video/video-material.tsx:260`, the `displayTexture` state is declared:

```typescript
const [displayTexture, setDisplayTexture] = useState(thumbnailTexture);
```

**The Issue**: `useState(thumbnailTexture)` captures the **initial value** of `thumbnailTexture` at the time of component render. Since `thumbnailTexture` starts as `null` (it's loaded asynchronously), `displayTexture` initializes to `null`. This happens before any effects run, including the thumbnail loading effect.

### The displayTexture Update Effect Logic Gap

The effect at `src/components/r3f/blocks/video/video-material.tsx:260-271` manages texture switching:

```typescript
useEffect(() => {
  if (texture) {
    setDisplayTexture(texture);
    if (thumbnailTexture) {
      thumbnailTexture.dispose();
      setThumbnailTexture(null);
    }
  } else if (thumbnailTexture) {
    setDisplayTexture(thumbnailTexture);
  }
}, [texture, thumbnailTexture]);
```

**The Logic Flow**:
1. Line 262: If `texture` (video) exists, use it and dispose thumbnail
2. Line 268: Otherwise, if `thumbnailTexture` exists, use it
3. **Missing**: No handling when both are `null` - the effect runs but doesn't call `setDisplayTexture`

### Race Condition in State Updates

The video loading effect at lines 116-232 updates multiple states when the video loads:

```typescript
// Line 172-174
setTexture(result.texture);     // Update 1
videoRef.current = result.video;
setIsLoading(false);            // Update 2
```

React batches these state updates. The component re-renders with the new `texture` and `isLoading` values, but the `displayTexture` effect hasn't run yet. The render condition at line 273 checks:

```typescript
if (isLoading || !displayTexture) {
  return <SkeletonMaterial />;
}
```

Since `isLoading` is now `false` but `displayTexture` is still `null` (effect pending), the component shows the skeleton instead of the video.

### How the Shader Receives Textures

The shader material at lines 285-331 receives the texture through uniforms:

```typescript
uniforms={{
  map: { value: displayTexture },
  radiusX: { value: BLOCK_BORDER_RADIUS / displayWidth },
  radiusY: { value: BLOCK_BORDER_RADIUS / displayHeight },
}}
```

The `map` uniform directly references `displayTexture`. React Three Fiber's reconciliation mechanism:
1. Detects when the `uniforms` prop changes (new object on every render)
2. Diffs the uniform values
3. Updates `material.uniforms.map.value` when `displayTexture` changes
4. Three.js automatically rebinds the texture during the next render pass

### The Complete Data Flow

#### Phase 1: Initial State
- `texture`: `null`
- `thumbnailTexture`: `null`
- `displayTexture`: `null` (from `useState(thumbnailTexture)`)
- Renders: SkeletonMaterial

#### Phase 2: Thumbnail Loads
- Thumbnail loads asynchronously at line 92-97
- `setThumbnailTexture(loadedTexture)` called
- Effect runs: `texture` is `null`, `thumbnailTexture` is set
- Line 269: `setDisplayTexture(thumbnailTexture)`
- Shader receives thumbnail texture

#### Phase 3: Video Starts Loading
- User hovers/selects, triggering `shouldLoadVideo = true`
- Video loading begins (worker or fallback path)

#### Phase 4: Video Loads (The Problem State)
- Video texture ready, `setTexture(result.texture)` called (line 172)
- `setIsLoading(false)` called (line 174)
- React batches updates and schedules render
- **Component renders BEFORE display effect runs**
- At this point:
  - `texture` = video texture (new value)
  - `isLoading` = false (new value)
  - `displayTexture` = thumbnail texture (OLD VALUE - effect hasn't run)
- Shader still has `map: { value: thumbnailTexture }` in uniforms
- Effect then runs and updates `displayTexture`, but render already happened

### Why It Works When Thumbnail Is Removed

When the thumbnail texture is completely removed (not loaded), the flow changes:
1. `thumbnailTexture` stays `null`
2. Video loads, `texture` becomes non-null
3. Effect runs with `texture` set and `thumbnailTexture` null
4. Line 263: `setDisplayTexture(texture)`
5. No thumbnail disposal needed (lines 264-266 skipped)
6. Shader receives video texture directly

## Code References

- `src/components/r3f/blocks/video/video-material.tsx:260` - displayTexture state initialization issue
- `src/components/r3f/blocks/video/video-material.tsx:261-271` - Effect with logic gap
- `src/components/r3f/blocks/video/video-material.tsx:172-174` - Video texture state updates
- `src/components/r3f/blocks/video/video-material.tsx:323-327` - Shader uniform binding
- `src/components/r3f/blocks/video/video-material.tsx:273-282` - Render condition checking displayTexture
- `src/components/r3f/video-block-result.tsx:69-72` - shouldLoadVideo trigger logic
- `src/lib/videos/video-texture-loader.ts:339-382` - Video texture loading coordination
- `src/lib/videos/helpers.ts:190-213` - getVideoThumbnailUrl helper

## Architecture Documentation

### Video Loading Pipeline

1. **Lazy Loading Trigger** (`video-block-result.tsx:69-72`):
   - `shouldLoadVideo` starts as `false`
   - Becomes `true` when user hovers or selects
   - Once true, never returns to false (one-way gate)

2. **Thumbnail Loading** (`video-material.tsx:85-114`):
   - Loads immediately on mount if `thumbnailUrl` provided
   - Uses THREE.TextureLoader
   - Configures texture with `configureTexture()` helper

3. **Video Loading** (`video-material.tsx:116-232`):
   - Triggered when `shouldLoadVideo` becomes `true`
   - Two paths: Worker (default) or Fallback
   - Worker path uses `videoCoordinator.loadVideoTexture()`
   - Creates THREE.VideoTexture on main thread (textures can't cross thread boundaries)

4. **Texture Switching** (`video-material.tsx:260-271`):
   - Priority: video texture > thumbnail texture
   - Disposes thumbnail when video loads
   - Updates shader uniform via React Three Fiber

### Key Implementation Details

The `VideoMaterial` component manages three texture states:
- `texture`: The video texture from worker/fallback loading
- `thumbnailTexture`: The static thumbnail image texture
- `displayTexture`: What's actually rendered (abstraction layer)

The shader (`video-material.tsx:293-322`) implements rounded corners with a custom fragment shader that samples the `map` uniform (bound to `displayTexture`).

React Three Fiber handles uniform updates through its reconciliation mechanism - when `displayTexture` state changes, it updates `material.uniforms.map.value` automatically.

## Historical Context

Based on the commit message "work in progress - video lazy loading" (d21bc5174), this is an in-progress implementation of lazy loading for videos with thumbnail placeholders. The feature aims to improve performance by:
1. Showing lightweight thumbnails immediately
2. Loading full videos only when user interacts
3. Seamlessly transitioning from thumbnail to video

## Related Research

- Previous video loading implementations may exist in earlier commits
- The video worker system (`video-texture-loader.ts`) is a sophisticated off-thread loading mechanism
- The pattern of thumbnail-to-full content switching is common in media-heavy applications

## Open Questions

1. Should the `displayTexture` state initialization be changed to handle the null case differently?
2. Is the effect dependency array complete, or are there missing dependencies that could cause stale closures?
3. Should the render condition at line 273 check for `texture` instead of `displayTexture` to avoid the race condition?
4. Would using `useLayoutEffect` instead of `useEffect` for the display texture update help ensure synchronous updates?