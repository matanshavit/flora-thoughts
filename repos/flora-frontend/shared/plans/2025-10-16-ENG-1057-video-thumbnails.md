# Video Thumbnails for Lazy-Loaded Blocks Implementation Plan

## Overview

Implement full-quality first-frame video thumbnails using ImageKit to replace blank skeleton placeholders that appear before videos lazy-load in the R3F canvas.

## Current State Analysis

After the lazy loading implementation (commit a4d10ca92), video blocks show gray pulsating skeleton placeholders until users interact (hover/select). Videos don't load at all until first interaction, creating a poor visual experience with blank tiles. While the `VideoUrlOutput` type includes a `thumbnailUrl` field, it's never populated during video generation or display.

### Key Discoveries:
- `VideoMaterial` currently shows `SkeletonMaterial` when no video texture exists (`video-material.tsx:211-220`)
- ImageKit supports video thumbnails via `/ik-thumbnail.jpg` suffix pattern
- Image nodes successfully use `toHumaneSizedImageUrl()` for thumbnail optimization
- The `thumbnailUrl` field exists in `VideoUrlOutput` type but is unused
- Videos use true lazy loading - only load when `shouldLoad = hasInteracted || selected || isHovered`

## Desired End State

Video blocks display high-quality thumbnail images immediately on render, showing the first frame of the video content. These thumbnails remain visible until the user interacts with the node (hover/select), triggering the full video load. The implementation reuses existing patterns from image nodes to avoid code duplication.

### Verification:
- Video blocks show thumbnail images instead of skeleton placeholders
- Thumbnails use full quality first frame without resolution/size changes
- Full video loads on first interaction as before
- Performance metrics remain unchanged (lazy loading benefits preserved)

## What We're NOT Doing

- NOT changing the lazy loading behavior (videos still load only on interaction)
- NOT storing thumbnails in the database or during generation
- NOT creating new components (reuse existing patterns)
- NOT changing video aspect ratio detection logic
- NOT implementing thumbnail caching separately from existing texture caching
- NOT modifying 2D canvas video rendering (focus on R3F only)

## Implementation Approach

Generate ImageKit thumbnail URLs dynamically from video URLs using the `/ik-thumbnail.jpg` pattern. Load thumbnail textures as a lightweight preview that displays immediately, then seamlessly transition to full video when the user interacts with the node.

## Phase 1: Thumbnail URL Generation Helper

### Overview

Create a utility function to generate ImageKit thumbnail URLs from video URLs, following existing patterns.

### Changes Required:

#### 1. Add Thumbnail URL Helper

**File**: `src/lib/videos/helpers.ts`
**Changes**: Add new helper function after existing video helpers

```typescript
/**
 * Generate an ImageKit thumbnail URL from a video URL
 * Uses the first frame (no time offset) at full quality
 * @param videoUrl - The video URL (must be ImageKit)
 * @returns Thumbnail URL or null if not an ImageKit video
 */
export function getVideoThumbnailUrl(videoUrl: string | null | undefined): string | null {
  if (!videoUrl) return null;

  // Only process ImageKit URLs
  if (!videoUrl.startsWith("https://ik.imagekit.io/")) {
    return null;
  }

  // Check if it's actually a video URL
  const isVideo = videoUrl.match(/\.(mp4|webm|mov)(\?|$)/i);
  if (!isVideo) return null;

  // Strip existing query parameters
  const cleanUrl = videoUrl.split("?")[0];

  // Append ImageKit thumbnail suffix for first frame at full quality
  // No transformations applied to maintain full quality
  return `${cleanUrl}/ik-thumbnail.jpg`;
}
```

### Success Criteria:

#### Automated Verification:
- [x] Type checking passes: `pnpm typecheck`
- [x] Linting passes: `pnpm lint`
- [x] Build succeeds: `pnpm build`

#### Manual Verification:
- [ ] Helper returns correct thumbnail URLs for ImageKit videos
- [ ] Helper returns null for non-ImageKit URLs
- [ ] Helper returns null for non-video URLs

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 2: Thumbnail Texture Loading

### Overview

Implement thumbnail texture loading in VideoMaterial component using Three.js TextureLoader, following the pattern from image components.

### Changes Required:

#### 1. Add Thumbnail Loading State

**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Add thumbnail loading logic after existing state declarations

```typescript
// Add after line 39 (existing state)
const [thumbnailTexture, setThumbnailTexture] = useState<THREE.Texture | null>(null);
const thumbnailLoaderRef = useRef<THREE.TextureLoader | null>(null);

// Add after line 60 (mounted effect)
// Initialize texture loader for thumbnails
useEffect(() => {
  thumbnailLoaderRef.current = new THREE.TextureLoader();
  return () => {
    thumbnailLoaderRef.current = null;
  };
}, []);
```

#### 2. Load Thumbnail Texture

**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Add thumbnail loading effect before the main video loading effect

```typescript
// Add new effect after line 60, before the main video loading effect
useEffect(() => {
  if (!videoUrl || !thumbnailLoaderRef.current) {
    setThumbnailTexture(null);
    return;
  }

  // Generate thumbnail URL using the helper
  const thumbnailUrl = getVideoThumbnailUrl(videoUrl);
  if (!thumbnailUrl) {
    return;
  }

  let disposed = false;

  // Load thumbnail texture
  thumbnailLoaderRef.current.load(
    thumbnailUrl,
    (loadedTexture) => {
      if (!disposed && mounted.current) {
        // Configure texture for video display
        loadedTexture.wrapS = THREE.ClampToEdgeWrapping;
        loadedTexture.wrapT = THREE.ClampToEdgeWrapping;
        loadedTexture.minFilter = THREE.LinearFilter;
        loadedTexture.magFilter = THREE.LinearFilter;
        loadedTexture.generateMipmaps = false;

        setThumbnailTexture(loadedTexture);
      }
    },
    undefined,
    (error) => {
      log.warn("Failed to load video thumbnail", {
        error: error instanceof Error ? error.message : String(error),
        videoUrl,
        thumbnailUrl,
      });
    }
  );

  return () => {
    disposed = true;
    if (thumbnailTexture) {
      thumbnailTexture.dispose();
    }
  };
}, [videoUrl]);
```

#### 3. Import Helper Function

**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Add import at the top of the file

```typescript
// Add after line 3
import { getVideoThumbnailUrl } from "@/lib/videos/helpers";
```

### Success Criteria:

#### Automated Verification:
- [x] Type checking passes: `pnpm typecheck`
- [x] Build succeeds: `pnpm build`
- [x] No TypeScript errors in VideoMaterial

#### Manual Verification:
- [ ] Thumbnail textures load successfully from ImageKit
- [ ] No console errors during thumbnail loading
- [ ] Textures are properly configured for display

---

## Phase 3: Thumbnail Display Integration

### Overview

Modify the VideoMaterial render logic to display thumbnail texture before video loads, replacing the skeleton placeholder.

### Changes Required:

#### 1. Update Render Logic

**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Modify the return statement to show thumbnail when available

```typescript
// Replace lines 211-220 with:
if (!videoUrl || (isLoading && !thumbnailTexture) || (!texture && !thumbnailTexture)) {
  return (
    <SkeletonMaterial
      width={displayWidth}
      height={displayHeight}
      radiusX={BLOCK_BORDER_RADIUS}
      radiusY={BLOCK_BORDER_RADIUS}
    />
  );
}

// Replace lines 222-270 with:
// Use thumbnail texture if video isn't loaded yet, otherwise use video texture
const displayTexture = texture || thumbnailTexture;

return (
  <shaderMaterial
    vertexShader={`
      varying vec2 vUv;
      void main() {
        vUv = uv;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
      }
    `}
    fragmentShader={`
      uniform sampler2D map;
      uniform float radiusX;
      uniform float radiusY;

      varying vec2 vUv;

      void main() {
        vec2 uv = vUv;
        vec2 center = vec2(0.5, 0.5);
        vec2 dist = abs(uv - center);

        // Check if we're in the rounded corner area
        bool inCorner = dist.x > 0.5 - radiusX && dist.y > 0.5 - radiusY;

        float alpha = 1.0;
        // Calculate normalized distance in corner
        if (inCorner) {
          vec2 corner = dist - vec2(0.5 - radiusX, 0.5 - radiusY);
          vec2 normCorner = vec2(corner.x / radiusX, corner.y / radiusY);
          float normDist = length(normCorner);

          // Smooth alpha falloff instead of hard discard (better GPU performance)
          alpha *= 1.0 - smoothstep(0.99, 1.01, normDist);
        }

        vec4 texColor = texture2D(map, uv);
        gl_FragColor = texColor * alpha;
      }
    `}
    uniforms={{
      map: { value: displayTexture },
      radiusX: { value: BLOCK_BORDER_RADIUS / displayWidth },
      radiusY: { value: BLOCK_BORDER_RADIUS / displayHeight },
    }}
    transparent
    side={THREE.DoubleSide}
    toneMapped={false}
  />
);
```

### Success Criteria:

#### Automated Verification:
- [x] Type checking passes: `pnpm typecheck`
- [x] Linting passes: `pnpm lint`
- [x] Build succeeds: `pnpm build`
- [x] Development server runs without errors: `pnpm dev`

#### Manual Verification:
- [ ] Thumbnails display instead of skeleton placeholders
- [ ] Thumbnails show first frame of video at full quality
- [ ] Smooth transition from thumbnail to video on interaction
- [ ] No flickering or visual artifacts during transition
- [ ] Lazy loading still works (videos only load on interaction)

---

## Phase 4: Cleanup and Optimization

### Overview

Add proper cleanup for thumbnail textures and ensure smooth transitions.

### Changes Required:

#### 1. Dispose Thumbnail on Video Load

**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Add thumbnail disposal when video texture is ready

```typescript
// Modify the video loading effect (around line 126) to dispose thumbnail when video loads:
// After line 126 where texture is set, add:
if (thumbnailTexture) {
  thumbnailTexture.dispose();
  setThumbnailTexture(null);
}
```

#### 2. Clean Up on Unmount

**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Ensure thumbnail is disposed on component unmount

```typescript
// Modify the cleanup return function (around line 165-172) to include thumbnail:
return () => {
  disposed = true;
  if (videoRef.current) {
    videoRef.current.pause();
  }
  if (thumbnailTexture) {
    thumbnailTexture.dispose();
  }
  // Clean up resources when component unmounts
  videoCoordinator.dispose(blockId);
};
```

### Success Criteria:

#### Automated Verification:
- [x] Type checking passes: `pnpm typecheck`
- [x] Build succeeds: `pnpm build`
- [ ] No memory leaks in browser DevTools

#### Manual Verification:
- [ ] Textures are properly disposed when switching between videos
- [ ] No memory accumulation when navigating between nodes
- [ ] Performance metrics remain stable during extended use

**Implementation Note**: After completing this phase and all verification passes, the implementation is complete.

---

## Testing Strategy

### Unit Tests:
- Test `getVideoThumbnailUrl()` helper with various URL formats
- Verify thumbnail URL generation for ImageKit videos
- Test null returns for non-ImageKit and non-video URLs

### Integration Tests:
- Verify thumbnail loads before video interaction
- Test transition from thumbnail to video on hover
- Verify lazy loading still prevents video load until interaction

### Manual Testing Steps:
1. Open a project with video nodes
2. Verify thumbnails appear immediately (no skeleton)
3. Hover over video node and verify video loads
4. Check that thumbnail quality matches first frame of video
5. Test with multiple video formats (mp4, webm, mov)
6. Verify performance with many video nodes on canvas
7. Check browser DevTools for memory leaks

## Performance Considerations

- Thumbnails are lightweight static images (~50-200KB vs full videos)
- Three.js TextureLoader handles caching automatically
- Thumbnail textures disposed when video loads to free memory
- No impact on lazy loading benefits (videos still load on demand)
- ImageKit CDN ensures fast thumbnail delivery

## Migration Notes

No migration needed - this is a pure enhancement that doesn't change data structures or require database updates. The feature will automatically apply to all existing video nodes.

## References

- Original research: `thoughts/shared/research/2025-10-16-ENG-1057-video-thumbnails-imagekit.md`
- ImageKit documentation: https://imagekit.io/docs/create-video-thumbnails
- Current video implementation: `src/components/r3f/video-block-result.tsx`
- Image thumbnail pattern: `src/components/ui/thumbnail-image.tsx`