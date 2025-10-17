# Video Thumbnail Display for Lazy-Loaded Video Blocks Implementation Plan

## Overview

Enhance the video lazy loading implementation from PR 2075 to display video thumbnails immediately on page load while maintaining the lazy loading behavior for actual video files. This addresses the requirement that video blocks should show their thumbnail image instead of a skeleton placeholder before user interaction.

## Current State Analysis

The current implementation (PR 2075) successfully lazy-loads videos, but displays a `SkeletonMaterial` (pulsating gray placeholder) until the user interacts with the block. Videos only load when `hasInteracted || selected || isHovered` becomes true. While this provides excellent performance benefits, it doesn't give users a visual preview of the video content.

### Key Discoveries:

- ImageKit infrastructure exists for generating video thumbnails: `videoUrl + "/ik-thumbnail.jpg"` (shared/files/thumbnails.ts:7-23)
- The `useOffThreadImageTextureLoader` hook can load images as textures off the main thread (src/components/r3f/hooks/use-offthread-texture-loader.tsx:12-98)
- Image blocks already implement a similar pattern of loading textures immediately (src/components/r3f/image-block-result.tsx:158-224)
- The `VideoUrlOutput` type already includes an optional `thumbnailUrl` field (src/components/nodes/types.ts:88-92)

## Desired End State

After implementation, video blocks will:

1. Display their thumbnail image immediately on page load (no skeleton)
2. Load the full video only when the user hovers or selects the block
3. Smoothly transition from thumbnail to video when the video loads
4. Fall back to skeleton only if the thumbnail fails to load

### Success Verification:

- Open a canvas with multiple video blocks
- All videos show thumbnails immediately (not skeletons)
- Network tab shows thumbnail requests but no video requests on initial load
- Hovering/selecting a block triggers video download
- Visual transition from thumbnail to video is smooth

## What We're NOT Doing

- Not changing the lazy loading behavior for videos
- Not modifying the Web Worker loading infrastructure
- Not changing how videos play/pause or handle muting
- Not altering the aspect ratio detection logic
- Not implementing thumbnail caching separately from the existing texture cache
- Not handling non-ImageKit videos (they'll continue showing skeleton)

## Implementation Approach

Use a two-stage texture loading approach:

1. **Stage 1**: Load thumbnail texture immediately using `useOffThreadImageTextureLoader`
2. **Stage 2**: Load video texture on interaction using existing lazy loading logic
3. Display thumbnail texture until video texture is ready, then swap seamlessly

## Phase 1: Add Thumbnail Loading to VideoMaterial Component

### Overview

Modify the VideoMaterial component to accept and load a thumbnail URL, displaying it while the video is not yet loaded.

### Changes Required:

#### 1. Update VideoMaterial Props and State

**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Add thumbnail support to the component

```typescript
// Add to VideoMaterialProps interface (around line 11)
interface VideoMaterialProps {
  videoUrl: string | undefined;
  thumbnailUrl?: string; // ADD THIS
  displayWidth: number;
  displayHeight: number;
  shouldPlay: boolean;
  blockId: string;
  quality?: "low" | "medium" | "high";
  useWorker?: boolean;
  muted?: boolean;
  onVideoLoaded?: (video: HTMLVideoElement) => void;
}

// Update component signature (around line 27)
export function VideoMaterial({
  videoUrl,
  thumbnailUrl, // ADD THIS
  displayWidth,
  displayHeight,
  shouldPlay,
  blockId,
  quality = "medium",
  useWorker = true,
  muted = true,
  onVideoLoaded,
}: VideoMaterialProps) {
```

#### 2. Add Thumbnail Texture Loading Hook

**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Import and use the image texture loader

```typescript
// Add import at the top (around line 2)
import { useOffThreadImageTextureLoader } from "@/components/r3f/hooks/use-offthread-texture-loader";

// Add thumbnail state and loading logic (after line 39)
const [thumbnailTexture, setThumbnailTexture] = useState<THREE.Texture | null>(null);
const [thumbnailLoaded, setThumbnailLoaded] = useState(false);
const { loadTexture } = useOffThreadImageTextureLoader();

// Add thumbnail loading effect (after line 60)
useEffect(() => {
  if (!thumbnailUrl) return;

  let isCancelled = false;

  const cancelLoad = loadTexture(
    thumbnailUrl,
    ({ texture }) => {
      if (!isCancelled && mounted.current) {
        // Configure texture for video display
        texture.wrapS = THREE.ClampToEdgeWrapping;
        texture.wrapT = THREE.ClampToEdgeWrapping;
        texture.minFilter = THREE.LinearFilter;
        texture.magFilter = THREE.LinearFilter;
        texture.generateMipmaps = false;

        setThumbnailTexture(texture);
        setThumbnailLoaded(true);
      }
    },
    (error) => {
      log.error("Failed to load video thumbnail", {
        error: error.message,
        blockId,
        thumbnailUrl,
      });
    },
  );

  return () => {
    isCancelled = true;
    cancelLoad();
    if (thumbnailTexture) {
      thumbnailTexture.dispose();
    }
  };
}, [thumbnailUrl, blockId, loadTexture]);
```

#### 3. Update Render Logic to Use Thumbnail

**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Modify the return statement to prioritize textures correctly

```typescript
// Update the render logic (around line 185)
// Determine which texture to display
const displayTexture = texture || thumbnailTexture;
const showSkeleton = !videoUrl || (!displayTexture && !thumbnailLoaded);

if (showSkeleton) {
  return (
    <SkeletonMaterial
      width={displayWidth}
      height={displayHeight}
      radiusX={BLOCK_BORDER_RADIUS}
      radiusY={BLOCK_BORDER_RADIUS}
    />
  );
}

if (!displayTexture) {
  // Thumbnail is loading but not ready yet
  return (
    <SkeletonMaterial
      width={displayWidth}
      height={displayHeight}
      radiusX={BLOCK_BORDER_RADIUS}
      radiusY={BLOCK_BORDER_RADIUS}
    />
  );
}

return (
  <shaderMaterial
    vertexShader={/* existing vertex shader */}
    fragmentShader={/* existing fragment shader */}
    uniforms={{
      map: { value: displayTexture }, // Use displayTexture instead of texture
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
- [x] Build succeeds: `pnpm build`
- [x] Unit tests pass: `pnpm test:unit`
- [x] Linting passes: `pnpm lint`

#### Manual Verification:

- [ ] VideoMaterial component accepts thumbnailUrl prop
- [ ] Thumbnail texture loads when provided
- [ ] Component displays thumbnail while video is not loaded
- [ ] No TypeScript errors in the component

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 2: Generate and Pass Thumbnail URLs

### Overview

Generate thumbnail URLs from video URLs and pass them to the VideoMaterial component from parent components.

### Changes Required:

#### 1. Add Thumbnail Generation to VideoBlockResultR3F

**File**: `src/components/r3f/video-block-result.tsx`
**Changes**: Import thumbnail utilities and generate thumbnail URL

```typescript
// Add import at the top (around line 2)
import { getOutputThumbnailUrlWithDimensions } from "shared/files/thumbnails";

// Generate thumbnail URL (after line 89, before skeletonMaterial)
const thumbnailUrl =
  videoUrl && videoUrl.includes("ik.imagekit.io")
    ? getOutputThumbnailUrlWithDimensions(
        videoUrl,
        Math.round(displayWidth * 2), // 2x for retina displays
        Math.round(displayHeight * 2),
      )
    : undefined;
```

#### 2. Pass Thumbnail URL to VideoMaterial

**File**: `src/components/r3f/video-block-result.tsx`
**Changes**: Add thumbnailUrl prop to VideoMaterial component

```typescript
// Update VideoMaterial usage (around line 124)
<VideoMaterial
  videoUrl={shouldLoad ? videoUrl : undefined}
  thumbnailUrl={thumbnailUrl} // ADD THIS
  displayWidth={displayWidth}
  displayHeight={displayHeight}
  shouldPlay={shouldPlay}
  blockId={nodeId}
  quality="medium"
  useWorker={true}
  muted={muted}
  onVideoLoaded={handleVideoLoaded}
/>
```

#### 3. Add Thumbnail Support to Static Video Block

**File**: `src/components/r3f/static-video-block-r3f.tsx`
**Changes**: Pass thumbnail URL through to VideoBlockResultR3F

Since StaticVideoBlockR3F passes videoUrl directly to VideoBlockResultR3F, the thumbnail generation will happen automatically in VideoBlockResultR3F. No changes needed here.

#### 4. Add Thumbnail Support to Generated Video Block

**File**: `src/components/r3f/video-block.tsx`
**Changes**: Verify that it also uses VideoBlockResultR3F

```typescript
// Verify the component uses VideoBlockResultR3F (should be around line 669)
// If it does, no changes needed as thumbnail generation happens in VideoBlockResultR3F
```

### Success Criteria:

#### Automated Verification:

- [x] Type checking passes: `pnpm typecheck`
- [x] All components compile without errors
- [x] Build succeeds: `pnpm build`

#### Manual Verification:

- [ ] Thumbnail URLs are correctly generated for ImageKit videos
- [ ] Thumbnail URLs include proper size parameters
- [ ] Non-ImageKit videos return undefined for thumbnail URL
- [ ] Props are correctly passed through component hierarchy

---

## Phase 3: Integration Testing and Refinement

### Overview

Test the complete implementation and refine any edge cases or visual issues.

### Changes Required:

#### 1. Handle Texture Disposal Properly

**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Ensure proper cleanup of both textures

```typescript
// Update the cleanup return in video loading effect (around line 139)
return () => {
  disposed = true;
  if (videoRef.current) {
    videoRef.current.pause();
  }
  // Clean up video texture but keep thumbnail
  videoCoordinator.dispose(blockId);
};

// Update the cleanup in thumbnail effect to not dispose if video is loaded
return () => {
  isCancelled = true;
  cancelLoad();
  // Only dispose thumbnail if video texture isn't loaded
  if (thumbnailTexture && !texture) {
    thumbnailTexture.dispose();
  }
};
```

#### 2. Add Loading State Tracking

**File**: `src/components/r3f/blocks/video/video-material.tsx`
**Changes**: Track whether we're transitioning between textures

```typescript
// Add state for tracking transitions (after line 39)
const [isTransitioning, setIsTransitioning] = useState(false);

// Update when video loads (in loadVideo function, around line 124)
setTexture(result.texture);
videoRef.current = result.video;
setIsTransitioning(false); // Video loaded, transition complete
setIsLoading(false);
```

### Success Criteria:

#### Automated Verification:

- [x] No memory leaks (textures properly disposed)
- [x] Type checking passes: `pnpm typecheck`
- [x] Linting passes: `pnpm lint`

#### Manual Verification:

- [ ] Open canvas with 10+ video blocks
- [ ] All videos show thumbnails immediately
- [ ] Network tab shows only thumbnail requests on load
- [ ] Hovering triggers video load for that block only
- [ ] Transition from thumbnail to video is smooth
- [ ] Moving between blocks doesn't cause flicker
- [ ] Thumbnails remain visible if video load fails
- [ ] Performance is acceptable with many videos

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Testing Strategy

### Unit Tests:

- Test thumbnail URL generation with various video URLs
- Test VideoMaterial with and without thumbnail URLs
- Verify texture disposal doesn't cause errors
- Test fallback behavior when thumbnail fails to load

### Integration Tests:

- Load canvas with mix of ImageKit and non-ImageKit videos
- Test interaction flows (hover, select, deselect)
- Verify network requests match expected pattern
- Test with slow network to verify progressive enhancement

### Manual Testing Steps:

1. Open dev tools Network tab filtered by Images and Media
2. Navigate to a canvas with multiple video blocks
3. Verify thumbnail images load immediately (jpg requests)
4. Verify no video files load initially (no mp4 requests)
5. Hover over a video block and verify:
   - Video file starts downloading
   - Thumbnail remains visible during download
   - Video replaces thumbnail when ready
6. Test with throttled network (Slow 3G):
   - Thumbnails should still load quickly
   - Videos should load progressively on interaction
7. Test error cases:
   - Block with invalid video URL (should show skeleton)
   - Block with non-ImageKit video (should show skeleton)

## Performance Considerations

- Thumbnail images are much smaller than videos (typically 10-50KB vs 1-10MB)
- Thumbnails load via ImageBitmapLoader off the main thread
- Maximum 2x resolution for retina displays (balance quality vs size)
- Existing texture caching prevents redundant loads
- Total bandwidth on initial load: ~500KB for 10 videos vs 50MB+ previously

## Migration Notes

No migration required. The changes are backward compatible:

- Existing video blocks will automatically show thumbnails
- Non-ImageKit videos will continue showing skeleton (current behavior)
- No data structure changes required

## References

- Original ticket: `thoughts/shared/tickets/ENG-1057.md`
- Related research: `thoughts/shared/research/2025-10-15-ENG-1057-video-thumbnail-loading-issue.md`
- Previous implementation: `thoughts/shared/plans/2025-01-14-ENG-1057-video-lazy-loading.md`
- Similar implementation: `src/components/r3f/image-block-result.tsx:158-224`
